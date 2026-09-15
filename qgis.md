# QGIS 道路資料處理

> 用 QGIS 搭配 QuickOSM 外掛，從 OpenStreetMap 撈出指定區域允許汽機車通行的道路，再加工成帶寬度的道路區塊面圖層。

## 目錄

- [📥 安裝 QGIS](#install)
- [🧩 安裝外掛 QuickOSM](#plugin)
- [🛣️ 查詢道路](#query)
  - [快速查詢](#quick-query)
  - [Overpass 自訂查詢](#overpass)
  - [查詢語法說明](#syntax)
- [💾 匯出結果](#export)
- [🛠️ 製作道路區塊](#block)
- [🚨 常見問題](#trouble)
- [📚 參考資料](#ref)

---

<a id="install"></a>

## 📥 安裝 QGIS

官方下載頁：<https://qgis.org/download/>

版本選擇：**LTR（Long Term Release）** 較穩定、外掛相容性好，一般用途優先選 LTR。

### Windows

- 官網下載「QGIS LTR」的獨立安裝檔（.msi）直接安裝。
- 或用 OSGeo4W 安裝器，可同時管理 GDAL、GRASS、SAGA 等工具，適合之後還要裝其他 GIS 套件。
- 也可用 winget：

  ```powershell
  winget install OSGeo.QGIS.LTR
  ```

### Ubuntu / Debian

```shell
sudo apt update
sudo apt install gnupg software-properties-common

# 匯入 QGIS 金鑰
sudo mkdir -m755 -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/qgis-archive-keyring.gpg https://download.qgis.org/downloads/qgis-archive-keyring.gpg

# 加入 LTR 套件庫
sudo tee /etc/apt/sources.list.d/qgis.sources > /dev/null <<'SRC'
Types: deb deb-src
URIs: https://qgis.org/ubuntu-ltr
Suites: $(lsb_release -cs)
Architectures: amd64
Components: main
Signed-By: /etc/apt/keyrings/qgis-archive-keyring.gpg
SRC

sudo apt update
sudo apt install qgis qgis-plugin-grass
```

> `Suites` 要填實際的發行版代號（如 `jammy`、`noble`），heredoc 內的 `$(lsb_release -cs)` 不會展開，請手動替換。

### WSL

QGIS 是 GUI 程式，WSL 內安裝需要 X Server 或 WSLg，設定方式見 [OpenCV / WSL GUI 顯示](opencv.md)。一般建議直接裝 Windows 版。

---

<a id="plugin"></a>

## 🧩 安裝外掛 QuickOSM

QuickOSM 透過 Overpass API 即時抓取 OSM 資料，不用先下載整份 .pbf。

1. 選單 **外掛程式 → 管理和安裝外掛程式**。
2. 搜尋 `QuickOSM` → 安裝。
3. 安裝後在 **向量 → QuickOSM** 或工具列出現綠色放大鏡圖示。

---

<a id="query"></a>

## 🛣️ 查詢道路

<a id="quick-query"></a>

### 快速查詢

適合單一條件，例如只抓次要道路：

1. 開啟 QuickOSM → **Quick query** 分頁。
2. 填入條件：

   | Key | Value | 說明 |
   |-----|-------|------|
   | `highway` | `secondary` | 次要道路 |
   | `lanes` | （留空） | 只要有 `lanes` 標籤的路，不限車道數 |

3. **In** 選「Layer extent」或輸入地名（如 `Taoyuan`）。
4. **Run query**，結果以圖層加入地圖。

常用 `highway` 值由大到小：`motorway`（高速公路）、`trunk`（快速道路）、`primary`（主要道路）、`secondary`（次要道路）、`tertiary`（三級道路）、`residential`（住宅區道路）、`service`（服務道路）。

<a id="overpass"></a>

### Overpass 自訂查詢

多條件、要排除禁行路段時，改用 **Query** 分頁貼上 Overpass QL：

```overpassql
[out:xml][timeout:25];

// 定位到桃園區域
{{geocodeArea:Taoyuan}} -> .area_0;

(
  // 允許汽車和機車通行的節點
  node
    ["highway"]
    ["access"!~"^(no|private)$"]
    ["motor_vehicle"!~"^(no)$"]
    ["motorcycle"!~"^(no)$"]
    (area.area_0);

  // 允許汽車和機車通行的道路
  way
    ["highway"~"^(motorway|trunk|primary|secondary|tertiary|motorway_link|trunk_link|primary_link|secondary_link|tertiary_link|residential|unclassified|living_street|service)$"]
    ["access"!~"^(no|private)$"]
    ["motor_vehicle"!~"^(no)$"]
    ["motorcycle"!~"^(no)$"]
    (area.area_0);

  // 允許汽車和機車通行的關聯
  relation
    ["highway"]
    ["access"!~"^(no|private)$"]
    ["motor_vehicle"!~"^(no)$"]
    ["motorcycle"!~"^(no)$"]
    (area.area_0);
);

// 連同組成道路的節點一起輸出
(._;>;);
out body;
```

同一段查詢也可以貼到 [Overpass Turbo](https://overpass-turbo.eu/) 線上預覽結果，再匯出 GeoJSON 拉進 QGIS。

<a id="syntax"></a>

### 查詢語法說明

| 語法 | 說明 |
|------|------|
| `[out:xml][timeout:25];` | 輸出 XML，逾時 25 秒。範圍大時可調高 timeout |
| `{{geocodeArea:Taoyuan}} -> .area_0;` | 以 Nominatim 地理編碼把地名轉成區域，存到變數 `area_0`。QuickOSM 與 Overpass Turbo 都支援此縮寫 |
| `node` / `way` / `relation` | OSM 三種元素：點、線（道路）、關聯（多段道路組成的路線） |
| `["highway"]` | 有 `highway` 標籤 |
| `["highway"~"^(a\|b)$"]` | `highway` 值符合正規表示式，`^...$` 精確比對 |
| `["access"!~"^(no\|private)$"]` | 排除 `access` 為 `no` 或 `private` 的禁行、私有道路 |
| `["motor_vehicle"!~"^(no)$"]` | 排除禁行汽車 |
| `["motorcycle"!~"^(no)$"]` | 排除禁行機車 |
| `(area.area_0)` | 限制在 `area_0` 範圍內 |
| `(._;>;);` | `._` 是目前結果集，`>` 遞迴取出其下的子元素（way 的節點），兩者合併，否則道路會沒有座標畫不出來 |
| `out body;` | 輸出完整標籤與座標 |

---

<a id="export"></a>

## 💾 匯出結果

查詢結果是暫存圖層，關閉 QGIS 就消失，需要另存：

1. 圖層面板右鍵 → **匯出 → 另存要素為**。
2. 格式常用：

   | 格式 | 用途 |
   |------|------|
   | GeoJSON | 程式讀取、網頁地圖（Leaflet / Mapbox） |
   | GeoPackage (.gpkg) | QGIS 內部使用，可含多圖層 |
   | Shapefile | 舊系統相容 |

3. CRS 一般選 `EPSG:4326`（WGS 84 經緯度）；台灣本地座標可選 `EPSG:3826`（TWD97 / TM2 zone 121）。

---

<a id="block"></a>

## 🛠️ 製作道路區塊

把 OSM 抓下來的道路中心線，加工成有寬度的面圖層。

```text
道路中心線  →  切割成小段  →  算出寬度  →  左右緩衝  →  合併成區塊
```

處理工具箱的快捷鍵是 `Ctrl + Alt + T`，以下工具都在裡面找得到。

### 1. 切割道路

先在交會處斷開，再切成等長的小段。

| 工具 | 用途 |
|------|------|
| Split with lines（用線分割） | 以圖層自身為切割線，在交匯點斷開 |
| Split Lines by Maximum Length（按最大長度分割線） | 把長線段切成不超過指定長度的小段 |

切完新增長度欄位，之後計算會用到：

| 欄位 | 表達式 |
|------|--------|
| `road_length` | `$length` |

> 欄位計算器算完要按**儲存編輯**，否則切換圖層時會遺失。

### 2. 計算道路寬度

OSM 的 `lanes` 欄位常常是空的，用 `coalesce` 給預設值，再乘上單一車道寬度：

| 欄位 | 表達式 |
|------|--------|
| `road_width` | `coalesce("lanes", 1) * 4` |

接著**另存為 `.shp`，座標參考系統選 `EPSG:3826`**（TWD97 / TM2 zone 121）。

> 這一步的座標系統很關鍵。緩衝區的距離單位跟著圖層的 CRS 走，用 `EPSG:4326` 的話 4 代表 4 度而不是 4 公尺，結果會離譜地大。

### 3. 建立左右緩衝區

用 **Buffer（單邊緩衝區）**，左右各跑一次：

| 參數 | 值 |
|------|-----|
| 距離 | 用表達式 `road_width / 2` |
| 邊 | 左 / 右，各做一次 |
| 線段 | `1` |
| 接點樣式 | 斜角（Bevel） |

> 線段設 1、接點用斜角，是為了避免轉彎處產生大量圓弧頂點。道路區塊只需要方正的形狀，頂點少後續處理也快。

### 4. 合併並標記方向

用 **Merge Vector Layers（合併向量圖層）** 把左右兩個緩衝圖層併成一個。

再新增方向角欄位，用來判斷道路走向：

| 欄位 | 表達式 |
|------|--------|
| `azimuth` | `degrees(azimuth(start_point($geometry), end_point($geometry)))` |

> `azimuth()` 回傳的是弧度，要用 `degrees()` 轉成 0 到 360 的角度才好判讀。

### 5. 處理路口

路口的區塊會互相重疊，需要另外併成一塊。原始筆記在這段較簡略，記錄用到的工具：

1. **DBSCAN 聚類** — 把鄰近的路口點分群
2. **Convex Hull（凸包）** — 每一群產生一個外框多邊形
3. **Multi-Ring Buffer（多環緩衝區）** — 環數 `1`，距離用表達式 `road_width / 2`
4. **Dissolve（融合）** — 進階參數要勾選「將不相交的要素分開」，否則全部會併成單一多部分幾何

### 6. 再切分區塊

區塊太大時需要依面積細分：

- **[Polygon Divider](https://github.com/jonnyhuck/RFCL-PolygonDivider)** — 第三方外掛，依指定面積把多邊形切開

也可以走線段的路線再轉回面：

1. 先把多邊形轉成線
2. **Split Lines by Maximum Length** 按最大長度分割
3. **Multipart Split**（多部分轉單一部分）

### 其他常用工具

| 工具 | 用途 |
|------|------|
| Densify Geometries | 在既有頂點之間插入更多頂點，讓後續的幾何運算更細緻 |
| `ST_MakeEnvelope`（PostGIS） | 用四個座標建立矩形邊界框，在資料庫端篩選範圍時使用 |

PostGIS 的空間查詢見 [PostgreSQL 筆記](postgresql.md)。

---

<a id="trouble"></a>

## 🚨 常見問題

- **查詢逾時或回傳空結果**：把 `timeout` 調大（如 90）、縮小區域，或改用 Quick query 的 Layer extent 限制範圍。
- **`{{geocodeArea:...}}` 找不到地名**：Nominatim 有時對中文或簡稱辨識不佳，改用英文全名（`Taoyuan City`），或先在 Overpass Turbo 測試。
- **道路畫不出來只有點**：查詢缺少 `(._;>;);`，way 沒有帶出組成節點。
- **Overpass API 429 Too Many Requests**：公開伺服器有速率限制，稍等再試，或在 QuickOSM 設定改用其他 Overpass 伺服器。

---

<a id="ref"></a>

## 📚 參考資料

- [QGIS 官方下載](https://qgis.org/download/)
- [QGIS Ubuntu 安裝說明](https://qgis.org/resources/installation-guide/#debian--ubuntu)
- [QuickOSM 原始碼與說明](https://github.com/3liz/QuickOSM)
- [Overpass QL 語法](https://wiki.openstreetmap.org/wiki/Overpass_API/Overpass_QL)
- [Overpass Turbo](https://overpass-turbo.eu/)
- [OSM Wiki：Key:highway](https://wiki.openstreetmap.org/wiki/Key:highway)
- [OSM Wiki：Key:access](https://wiki.openstreetmap.org/wiki/Key:access)
