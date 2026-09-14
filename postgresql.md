# PostgreSQL 筆記（道路巡查系統 web_server）

> 後端 `web_server` 的資料庫筆記。Schema 的唯一來源是 NestJS + TypeORM 的 entity 與 migration，本篇整理環境設定、設計慣例與資料表總覽，方便查找，不取代程式碼。

- Entity：`webSystem/my-app/server/src/<模組>/entities/*.entity.ts`
- Migration：`webSystem/my-app/server/src/migrations/`
- 資料庫容器：`postgres-db/`

> 早期 HackMD 筆記整理的舊版結構（大寫複數表名，與現行不符）保留在 [postgresql-legacy-schema.sql](postgresql-legacy-schema.sql)，僅供對照，不要拿去執行。

## 目錄

- [🐳 環境](#env)
- [🔐 pg_hba.conf 連線授權](#pg-hba)
- [⚙️ postgresql.conf 重點調校](#pg-conf)
- [🧩 擴展](#extensions)
- [🔄 Migration 機制](#migration)
- [🏗️ 設計慣例](#patterns)
  - [同時支援 PostgreSQL 與 MSSQL](#dual-db)
  - [分區表 + REF 參照表](#partition-ref)
  - [PostGIS 幾何欄位](#postgis)
  - [權限四層架構](#permission)
  - [共用欄位慣例](#common-columns)
- [🗂️ 資料表總覽](#tables)
- [📅 分區維護](#partition-maint)
- [📚 參考資料](#ref)

---

<a id="env"></a>

## 🐳 環境

| 項目 | 內容 |
|------|------|
| 映像檔 | `postgis/postgis:17-3.5`（PostgreSQL 17 + PostGIS 3.5） |
| 設定檔 | `postgres-db/config/postgresql.conf`、`pg_hba.conf`，建置時 `COPY` 進映像檔 |
| 資料保存 | named volume `postgres_data`，掛載於 `/var/lib/postgresql/data` |
| 帳密 / DB 名 | 根目錄 `.env` 的 `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` |

```shell
bash start.sh postgres-db              # 建置並啟動
bash start.sh postgres-db --no-build   # 用現有映像檔直接啟動
```

> 設定檔是 `COPY` 進映像檔而不是掛載，**改了 `config/` 底下的檔案一定要重新建置**，`--no-build` 會沿用舊映像檔裡的舊設定。

> `clean.sh` 不會刪 `postgres_data`。要真的清掉資料得手動 `docker volume rm`。

連線測試：

```shell
psql -h <host> -p <port> -U <user> -d <database>
docker exec postgres pg_isready -U <user>
```

---

<a id="pg-hba"></a>

## 🔐 pg_hba.conf 連線授權

實際清單在 `postgres-db/config/pg_hba.conf`，結構如下（IP 為示意，正式清單見該檔）：

```text
# TYPE  DATABASE      USER  ADDRESS            METHOD
local   all           all                      scram-sha-256
host    all           all   127.0.0.1/32       scram-sha-256
host    all           all   ::1/128            scram-sha-256

# 複寫
local   replication   all                      scram-sha-256
host    replication   all   127.0.0.1/32       scram-sha-256
host    replication   all   ::1/128            scram-sha-256
host    replication   all   172.23.0.0/16      scram-sha-256   # docker app-network

# 內網網段
host    all           all   172.16.0.0/12      scram-sha-256
host    all           all   192.168.0.0/16     scram-sha-256

# 指定的對外固定 IP，一行一個 /32
host    all           all   203.0.113.10/32    scram-sha-256
```

- 認證一律 `scram-sha-256`，搭配 `postgresql.conf` 的 `password_encryption = scram-sha-256`。
- `172.23.0.0/16` 是 docker compose 的 `app-network` 網段，容器之間靠這段互連。
- 連線被拒時先確認來源 IP 有沒有被涵蓋。

改完設定要重新載入：

```shell
docker exec postgres psql -U <user> -c "SELECT pg_reload_conf();"
# 或重新建置容器（設定是 COPY 進映像檔的）
bash start.sh postgres-db
```

---

<a id="pg-conf"></a>

## ⚙️ postgresql.conf 重點調校

針對大記憶體主機（容器 `mem_limit: 64g`、`shm_size: 4gb`）調過的參數：

| 參數 | 值 | 說明 |
|------|-----|------|
| `shared_buffers` | `16GB` | 約主機記憶體的 1/4 |
| `effective_cache_size` | `48GB` | 給查詢規劃器的快取估計值，非實際配置 |
| `work_mem` | `32MB` | 單一排序 / hash 作業的上限 |
| `maintenance_work_mem` | `1GB` | 建索引、VACUUM 用 |
| `max_connections` | `100` | 後端 `poolSize: 30` |
| `random_page_cost` | `1.1` | SSD 設定，預設 4.0 是機械硬碟 |
| `effective_io_concurrency` | `200` | SSD 可平行的 I/O 數 |
| `max_parallel_workers` | `12` | 單一查詢最多 `max_parallel_workers_per_gather = 4` |
| `wal_compression` | `zstd` | 壓縮 full-page write |
| `max_wal_size` | `8GB` | 搭配 `checkpoint_timeout = 15min` 降低 checkpoint 頻率 |
| `default_statistics_target` | `200` | 統計取樣加倍，改善大表的查詢計畫 |

後端另外設了逾時保護（`app.module.ts`）：

```text
statement_timeout                     = 300000   # 單一 SQL 5 分鐘
lock_timeout                          = 300000   # 等待資料列鎖 5 分鐘
idle_in_transaction_session_timeout   = 300000   # 交易中閒置 5 分鐘
```

---

<a id="extensions"></a>

## 🧩 擴展

由 migration `CreateExpansion` 建立：

```sql
CREATE EXTENSION IF NOT EXISTS postgis;   -- 地理空間
CREATE EXTENSION IF NOT EXISTS pg_trgm;   -- Trigram 索引，加速案件編號與地址的模糊搜尋
```

`pg_trgm` 實際用在 `work_order`、`maintenance` 的 `case_num` 與 `address` 欄位。

---

<a id="migration"></a>

## 🔄 Migration 機制

TypeORM 設定（`app.module.ts`）：

```text
synchronize:   false   # 正式環境務必關閉，避免資料表被覆蓋
migrationsRun: true    # 後端啟動時自動跑未執行的 migration
migrations:    migrations/**/*.{ts,js}
```

目錄分兩層：

| 目錄 | 用途 |
|------|------|
| `migrations/initialize/` | 建立各模組的初始結構，一個模組一個檔（`CreateCasePatrol`、`CreateOrgstruct` 等 21 個） |
| `migrations/update/<年份>/` | 之後的結構異動，依年份分資料夾 |

常用指令：

```shell
yarn migrate:create <Name>   # 在 update/<今年> 底下建立新的 migration 檔
yarn migrate:revert          # 回退最後一次 migration
yarn migrate:legacy          # 匯入舊系統資料
```

每個 migration 的 `up()` 會先判斷資料庫種類再分流：

```ts
const db = getDBType(qr);
if (db === 'postgres') await this.upPostgres(qr);
else if (db === 'mssql') await this.upMssql(qr);
else throw new Error(`Unsupported db: ${db}`);
```

---

<a id="patterns"></a>

## 🏗️ 設計慣例

<a id="dual-db"></a>

### 同時支援 PostgreSQL 與 MSSQL

同一份 entity 要能跑在兩種資料庫上，靠 `util/entity-columns.ts` 的自訂裝飾器依 `config.database.vendor` 切換型別：

| 裝飾器 | PostgreSQL | MSSQL |
|--------|------------|-------|
| `DbJsonColumn` | `jsonb` | `nvarchar(max)` + JSON transformer |
| `DbBooleanColumn` | `boolean` | `bit` |
| `DbGeomPointColumn` | `geometry(Point, <srid>)` | `geometry` + WKT transformer |

Migration 端則有 `util/migration-utils.ts` 的 `stringColumn`、`datetimeColumn`、`jsonColumn`、`floatColumn`、`boolColumn` 做同樣的事。

> 新增欄位時用這些包裝函式，不要直接寫 `@Column({ type: 'jsonb' })`，否則 MSSQL 會建不起來。

<a id="partition-ref"></a>

### 分區表 + REF 參照表

三張資料量最大的表依 `dt_record` 做季分區：`case_patrol`、`vehicle_track`、`case_survey`。

分區表有兩個限制，各自帶出一個解法：

1. **主鍵必須包含分區鍵** → 主鍵是 `(id, dt_record)` 複合鍵。
2. **其他表不能對分區表建外鍵** → 另建 `*_ref (id PK, dt_record)` 參照表，由觸發器與主表同步，所有子表外鍵指向 REF 表。

```text
case_patrol  (分區表, PK = id + dt_record)
     │  同步
     ▼
case_patrol_ref  (id PK)
     ▲
     │ FOREIGN KEY ... ON DELETE CASCADE
     ├── case_patrol_address          地址
     ├── case_patrol_status           篩選 / 編輯 / 需修復狀態
     ├── case_patrol_taipei           台北客製欄位
     ├── upload_status_case_patrol    各平台上傳狀態
     ├── work_order_patrol            派工單來源
     └── road_damage                  路況評估
```

`vehicle_track` 與 `case_survey` 同樣結構。

分區由 `ensureQuarterPartitions(qr, table, 2023, 今年 + 1)` 建立，PostgreSQL 會一併建 `<table>_default` 作為預設分區，落在範圍外的資料不會直接寫入失敗。

<a id="postgis"></a>

### PostGIS 幾何欄位

| SRID | 用途 |
|:----:|------|
| `4326` | WGS 84 經緯度，`gis_region.geom` 等來自外部的圖資 |
| `3826` | TWD97 / TM2 zone 121，系統內部計算與比對一律用這個 |

案件與軌跡表除了 `longitude` / `latitude` 兩個 double precision 欄位，另存一個 `geom_3826` 供空間查詢與 spatial index 使用。濱海（binhai）模組是例外，只存經緯度沒有幾何欄位。

<a id="permission"></a>

### 權限四層架構

```text
system  →  module  →  feature  →  feature_action
   系統       模組        子功能        操作（CREATE / DELETE ...）
```

公司端以三張表逐層開通：`company_module` → `company_module_feature` → `company_module_feature_action`。

使用者權限則是：角色 `role_feature_action` 給予基礎權限，再由 `user_action_override` 以 `is_granted` 個別加簽或撤銷。

<a id="common-columns"></a>

### 共用欄位慣例

| 欄位 | 說明 |
|------|------|
| `created_at` / `updated_at` | 由 TypeORM 的 `@CreateDateColumn` / `@UpdateDateColumn` 維護，不用資料庫觸發器 |
| `is_active` | 停用旗標，系統用停用取代刪除，沒有 `deleted_at` 軟刪除欄位 |
| `key` | 對外的字串代碼，通常有 unique 限制；`id` 是內部流水號 |
| `prj_id` | 標案的對外代碼字串，注意與 `project.id` 流水號不同，統計表多用 `prj_id` 分群 |
| `dt_record` | 資料發生時間，分區鍵；`created_at` 是寫入時間 |

---

<a id="tables"></a>

## 🗂️ 資料表總覽

共 102 張表。表名為 entity 的 `@Entity({ name })`。

### orgstruct 組織與權限

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `system` | 系統定義表，最上層的系統別 | `key` 唯一；一對多 `module` |
| `module` | 系統底下的功能模組，含圖示、路徑與排序 | `system_id` → `system`；unique (`system_id`,`key`) |
| `feature` | 模組底下的子功能（選單項目） | `module_id` → `module`；`key` 唯一 |
| `feature_action` | 子功能可執行的操作 | `feature_id` → `feature`；`key` 唯一，例 `ROAD_EVAL:CREATE` |
| `company` | 公司主檔，含公司編號、名稱與使用者上限 | `key`、`name` 各自唯一；`user_limit`、`is_active` |
| `department` | 公司部門結構，支援多層樹狀 | `company_id` → `company`；`parent_id` → 自身；unique (`company_id`,`code`) |
| `role` | 公司層級的角色定義 | `company_id` → `company`；unique (`company_id`,`name`)；`is_system` |
| `role_feature_action` | 角色與功能操作的授權對應 | `role_id`、`feature_action_id`；組合唯一 |
| `company_module` | 公司可使用哪些模組 | `company_id`、`module_id`；組合唯一，`is_active` |
| `company_module_feature` | 公司在該模組下可使用哪些子功能 | `company_module_id`、`feature_id`；組合唯一 |
| `company_module_feature_action` | 公司該子功能可使用哪些操作 | `company_module_feature_id`、`feature_action_id`；組合唯一 |

### auth 帳號

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `users` | 使用者主檔，含帳號、密碼雜湊、到期日與登入狀態 | `company_id`、`department_id`、`role_id`、`home_sys_id`、`manager_id` → 自身；unique (`key`)、(`company_id`,`account`) |
| `user_action_override` | 使用者個別的操作加簽或撤銷 | unique (`user_id`,`feature_action_id`)；`is_granted`、`reason` |
| `user_device` | 使用者裝置註冊，供推播使用 | `user_id` → `users`；unique (`user_id`,`platform`) |
| `password_history` | 歷史密碼紀錄，防止重複使用 | `user_id` → `users`；索引 (`user_id`,`created_at`) |
| `user_taipei_app` | 臺北磐碩 APP 的獨立帳號表 | `account` 唯一；與 `users` 無關聯 |

### project 標案

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `project` | 標案主表，含標案編號、名稱、業主與起迄期間 | unique (`prj_id`)、(`prj_no`)；`proprietor_level` |
| `company_project` | 標案與公司的對應，每標案限一家公司 | `company_id`、`project_id`；unique (`project_id`) |
| `section` | 工務段主檔，隸屬公司 | `company_id` → `company`；unique (`company_id`,`key`) |
| `project_section` | 標案與工務段的對應 | `project_id`、`section_id`；unique 兩者 |
| `section_area` | 工務段負責的行政區 | `project_section_id`、`area_id`；unique 兩者 |
| `area` | 縣市 / 鄉鎮市區行政區主檔 | unique (`county`,`district`) |
| `vehicle` | 車輛主檔 | `name` 唯一 |
| `project_vehicle` | 標案可使用的車輛 | `project_id`、`vehicle_id`；unique 兩者 |
| `dealer` | 資料上傳廠商主檔 | `key` 唯一 |
| `project_upload_dealer` | 標案與上傳廠商的授權對應 | `project_id`、`dealer_id`；unique 兩者 |
| `project_taipei_app` | 臺北 APP 專用的標案與行政區對應 | unique (`prj_id`,`district`) |

外鍵多為 `ON DELETE CASCADE`，關聯表一律帶 `is_active` 作停用旗標。

### case-patrol 車巡（AI 影像辨識）

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `case_patrol` | 車巡破壞案件主表，含裂縫類型、尺寸與座標 | **分區表**，PK (`id`,`dt_record`)；`project_vehicle_id`；unique (`dt_record`,`img_detect`,`crack_id`)；`geom_3826` |
| `case_patrol_ref` | 車巡案件參照表，供子表單鍵外鍵關聯 | PK `id`；(`id`,`dt_record`) → `case_patrol` |
| `case_patrol_address` | 案件的行政區與門牌地址 | `case_patrol_id` → `case_patrol_ref`（unique） |
| `case_patrol_status` | 篩選、編輯與需修復狀態及異動者 | `case_patrol_id` → `case_patrol_ref`（unique）；四個異動者欄位 → `users` |
| `case_patrol_taipei` | 臺北市專用的補充欄位 | `case_patrol_id` → `case_patrol_ref`（unique）；`main_type`、`dtype_code` |
| `upload_status_case_patrol` | 對各外部平台的上傳狀態與時間 | `case_patrol_id` → `case_patrol_ref`（unique）；`upload_panshuo` / `srgeo` / `bkup` / `txg` |
| `vehicle_track` | 車輛 GPS 軌跡點 | **分區表**，PK (`id`,`dt_record`)；unique (`dt_record`,`prj_id`,`car`)；`geom_3826` |
| `vehicle_track_ref` | 軌跡參照表 | PK `id`；(`id`,`dt_record`) → `vehicle_track` |
| `vehicle_track_address` | 軌跡點反查的行政區與道路名稱 | `vehicle_track_id` → `vehicle_track_ref`（unique） |
| `upload_status_vehicle_track` | 軌跡對外部平台的上傳狀態 | `vehicle_track_id` → `vehicle_track_ref`（unique） |
| `daily_check` | 每日每車的案件數量與上傳結果稽核 | unique (`date`,`project_vehicle_id`)；多個 JSONB 統計欄 |

### case-survey 鋪面人工調查

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `case_survey_order` | 調查工單（收件單） | unique `order_number`；`is_active` |
| `case_survey_order_detail` | 工單下各路段明細，含樣本數、車道方向與樁號 | `case_survey_order_id` → `case_survey_order` |
| `case_survey_report` | 各路段產出的報表檔案與進度 | `case_survey_order_detail_id`；unique `key`、(`detail_id`,`type`) |
| `case_survey` | 調查案件主表，含破壞類型、尺寸與調查人員 | **分區表**，PK (`id`,`dt_record`)；`order_number`；三個人員欄位 → `users` |
| `case_survey_ref` | 調查案件參照表 | PK `id`；(`id`,`dt_record`) → `case_survey` |
| `case_survey_address` | 道路位置、起迄點與路寬面積 | `case_survey_id` → `case_survey_ref`（unique） |
| `case_survey_status` | 檢查狀態與異動人員 | `case_survey_id` → `case_survey_ref`（unique） |

### work-order 巡查單與派工單

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `maintenance` | 巡查單 / 巡修單主表，記錄破壞位置與調查資訊 | `project_id`、`survey_user_id` → `users`；unique `case_num`；`geom_3826` |
| `maintenance_status` | 巡查單狀態與最後更新人員 | `maintenance_id`（unique）；`upd_status_usr` → `users` |
| `maintenance_repair` | 巡修單 (RB) 的材料與回填尺寸 | `maintenance_id`（unique）；`material`、`repair_date` |
| `maintenance_image` | 巡查單各類型照片路徑 | unique (`maintenance_id`,`img_type`) |
| `work_order` | 派工單主表，含施工地點、期程、材料與尺寸 | `project_id`；unique `case_num`；`start_geom_3826` / `end_geom_3826` |
| `work_order_status` | 派工單狀態（待處理 / 施工中 / 已回報 / 已完工） | `work_order_id`（unique）；`upd_status_usr` → `users` |
| `work_order_image` | 派工單各類型施工照片路徑 | unique (`work_order_id`,`img_type`) |
| `work_order_improvement` | 路基改善 (PB) 的取樣與試驗資料 | `work_order_id`（unique）；`test_item` JSONB |
| `work_order_maintenance` | 巡查單 → 派工單來源關聯 | `work_order_id`、`maintenance_id` 各自 unique |
| `work_order_patrol` | 車巡案件 → 派工單來源關聯 | `case_patrol_id` → `case_patrol_ref` |
| `work_order_user` | 派工單與施工人員指派 | unique (`work_order_id`,`user_id`) |

`case_num` 與 `address` 建有 trigram 索引供模糊搜尋。

### road-eval 路況評估

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `road_eval` | 各道路區塊每日的評估分數 | `project_vehicle_id`；unique (`date`,`project_vehicle_id`,`road_code`) |
| `road_damage` | 評估下的個別損壞案件與修復狀態 | `road_eval_id`、`case_patrol_id` → `case_patrol_ref` |
| `road_eval_calc` | 每日每車的評估案件數統計 | unique (`date`,`project_vehicle_id`) |

### geo 圖資

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `geo_area` | 縣市 / 鄉鎮 / 村里名稱代碼對照 | unique (`county_name`,`district_name`,`cavlge_name`) |
| `gis_region` | 行政區界圖資 | unique (`county_code`,`town_code`,`vill_code`)；`geom` (4326)、`geom_3826`、`geom_town` |
| `gis_road_layer_ntpc` | 新北市道路區塊圖層 | PK `fid`；`road_code`、`geom` (Polygon, 3826) |
| `gis_road_layer_tyn` | 桃園市道路區塊圖層 | 同上 |
| `gis_road_layer_hsc` | 新竹縣市道路區塊圖層 | 同上 |
| `gis_road_layer_txg` | 臺中市道路區塊圖層，含路線標記 | 同上，另有 `roadname`、`has_route` |
| `gis_road_meas` | 道路丈量長寬與面積 | unique (`county`,`district`,`road_name`,`length`,`width`)；`geom_3826` (MultiPolygon) |

四張 `gis_road_layer_*` 由抽象類別 `LayerBase` 繼承，是同結構的分縣市分表。

### patrol-route 巡掃路線

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `gis_route_txg` | 臺中市巡掃路線線段圖資 | `geom` (MultiLineString, 3826) |
| `txg_line` | 路線啟用狀態，`id` 對應 `gis_route_txg` | PK `id`（非自增）；`is_active` |
| `patrol_daily_stats_point` | 每日巡掃覆蓋率（當日與累計） | `record_date`、`prj_id`、`district`；`daily_rate`、`cumulative_rate`；JSONB 陣列欄位 |

### location 地理編碼

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `address_lookup` | 地址正規化與座標對照（地理編碼結果快取） | unique `o_address`；`n_address`、各層級行政區欄位、`geom_3826` |
| `address_grid` | 已處理的地址網格範圍 | unique (`grid_x`,`grid_y`) |

### case-app 臺北磐碩 APP

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `case_taipei_app` | APP 回報案件，含施工前中後照片 | unique `case_num`、`prj_code_num`；`dtype`、`degree` |
| `upload_status_taipei_app` | APP 案件的上傳狀態 | `case_taipei_app_id`（unique）；`upload_panshuo`、`upload_bkup` |

### binhai 濱海客製

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `binhai_case` | 道路問題事件主檔（無人機 / 巡查車辨識） | unique `case_num`、(`device_code`,`dt_record`)；`specialty_id`、`problem_type_id` |
| `binhai_case_problem` | 事件的標注問題明細（尺寸、嚴重度、標注框） | `binhai_case_id`；unique (`binhai_case_id`,`item_no`)；`mark_position` JSONB |
| `binhai_track` | 設備軌跡點（位置、高度、在線狀態） | unique (`device_code`,`dt_record`)；`online`、`status` JSONB |
| `upload_status_binhai_case` | 事件上傳市政平台的狀態與重試 | 1:1 `binhai_case_id`；`retry_count` |
| `upload_status_binhai_track` | 軌跡上傳市政平台的狀態與重試 | 1:1 `binhai_track_id`；`retry_count` |

座標只存 `longitude` / `latitude`，未使用 PostGIS。

### ai-studio AI 標注

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `ai_project` | AI 訓練專案，含任務型別與所用模型 | `model_id` → `ai_model`；`task_type`、`status` |
| `ai_model` | 可呼叫的模型清單與推論 API 位址 | `name`、`api` |
| `ai_dataset` | 資料集主檔，依任務型別分類 | `name`、`task_type` |
| `ai_image` | 待標注影像來源，含二篩案件溯源 | `case_patrol_id`；`is_annotated`、`is_active` |
| `ai_label` | 標注標籤字典（名稱、顏色、任務型別） | `name`、`task_type`、`color` |
| `ai_category` | 標注分類字典 | `name`、`remark` |
| `ai_task` | 標注派工單，指派影像批次給標注人員 | `assigned_to` / `assigned_by` → `users`；`image_count` |
| `ai_annotation` | 單張影像的標注紀錄與審核狀態 | `image_id` → `ai_image`；`annotation_data` JSONB；`status` |
| `ai_dataset_annotation` | 資料集內已定案的標注快照 | 複合 PK (`dataset_id`,`image_id`)；`annotation_data` JSONB |

### dashboard 統計快照

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `dashboard_case_patrol` | 車巡統計：里程、天數與各類破壞數量 | unique (`date`,`prj_id`,`county`,`district`) |
| `dashboard_work_order` | 派工統計：施工中、已回報、完工與逾期件數 | unique (`date`,`prj_id`,`county`,`district`) |

兩張都是預先彙總的快照表，以 `prj_id` 分群，無外鍵。

### report 報表

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `report` | 報表產製管理：期間、類型、狀態、進度與檔案路徑 | unique `key`；`type`、`status`、`progress`、`path` |
| `company_report` | 公司、使用者與報表的關聯 | `company_id`、`report_id`、`user_id`；unique (`company_id`,`report_id`) |

### core 字典表

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `announcement` | 系統公告 | `type`、`title`、`date` |
| `crack_type` | 各縣市破壞（裂縫）類型代碼 | unique (`county`,`key`) |
| `crack_type_survey` | 調查用破壞類型代碼，含計量單位 | unique `key`；`quantity` |
| `degree_type` | 各縣市損壞程度代碼 | unique (`county`,`key`) |
| `form_type` | 表單類型代碼（車巡單、巡查單等） | unique (`type`,`key`)；`is_active` |
| `image_type` | 影像用途類型代碼 | unique `type` |
| `status_type` | 各類流程狀態代碼，含顯示顏色 | unique (`type`,`key`)；`color`、`is_active` |

`crack_type`、`crack_type_survey`、`degree_type` 三個 entity 定義在同一個 `crack-type.entity.ts` 檔內。

### 其他

| 資料表 | 說明 | 主要欄位 / 關聯 |
|--------|------|------------------|
| `case_history` | 各類案件的版本化異動歷程，保存整筆快照 | 複合 PK (`case_type`,`case_id`,`version`)；`snapshot_json` JSONB；`modified_by` → `users` |
| `case_sequence` | 案件編號流水號產生器，依公司表單代號與日期累計 | 複合 PK (`bid_prefix`,`case_date`)；`last_number` |
| `audit_log` | 系統操作稽核，保存請求、前後值與錯誤 | `user_id` → `users`；`action`、`result`、`request_id`；多個 JSONB 欄位 |
| `vehicle_comm` | 巡查車連線狀態與最後回報位置 | PK `vehicle`；`is_open`、`stream_open`、`last_time`、`geom_3826` |

`case_history` 的 `case_type` + `case_id` 是跨表的多型參照，沒有實體外鍵。

---

<a id="partition-maint"></a>

## 📅 分區維護

`ensureQuarterPartitions` 在 migration 執行時會建到「今年 + 1」，所以每年只要後端有重新部署過就會自動補上隔年的分區。需要手動補時：

```sql
CREATE TABLE IF NOT EXISTS case_patrol_2027_q1 PARTITION OF case_patrol
  FOR VALUES FROM ('2027-01-01') TO ('2027-04-01');
-- vehicle_track、case_survey 同樣方式
```

確認分區現況：

```sql
-- 列出某分區表的所有子分區與範圍
SELECT c.relname, pg_get_expr(c.relpartbound, c.oid) AS bound
FROM pg_class p
JOIN pg_inherits i ON i.inhparent = p.oid
JOIN pg_class c ON c.oid = i.inhrelid
WHERE p.relname = 'case_patrol'
ORDER BY c.relname;

-- 檢查有沒有資料掉進 default 分區（代表缺對應季度的分區）
SELECT count(*) FROM case_patrol_default;
```

`<table>_default` 由 `ensureQuarterPartitions` 一併建立，超出範圍的資料會落在這裡而不是寫入失敗，所以要定期檢查它是不是空的。

---

<a id="ref"></a>

## 📚 參考資料

- [pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Server Configuration](https://www.postgresql.org/docs/current/runtime-config.html)
- [PostGIS](https://postgis.net/)
- [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)
- [TypeORM Migrations](https://typeorm.io/migrations)
