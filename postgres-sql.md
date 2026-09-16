# SQL 工作筆記整理

> 從 5 個雜亂的 scratch SQL 檔案整理而成，保留日後換工作/換專案還用得到的通用查詢範本、
> PostgreSQL/PostGIS 技巧與設計模式。公司內部特定的權限體系、標案、上傳狀態查詢已移除。
> 使用時請依實際 schema 版本調整欄位/資料表名稱。

---

## 1. PostgreSQL 常用管理 / 監控指令

```sql
-- 目前連線數 / 連線明細
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
SELECT usename, client_addr, datname, application_name, state, query 
FROM pg_stat_activity;

-- 資料表容量
SELECT pg_size_pretty(pg_total_relation_size('表名')) AS total_size;

-- log 相關設定
SHOW logging_collector;
SHOW log_destination;
SHOW log_directory;
SHOW log_filename;
SHOW log_statement;
ALTER SYSTEM SET log_statement = 'mod';   -- 記錄 insert/update/delete
SELECT pg_reload_conf();                  -- 重新載入設定（不需重啟）

-- 記憶體參數（依主機規格調整後套用）
-- ALTER SYSTEM SET shared_buffers = '32GB';
-- ALTER SYSTEM SET temp_buffers = '64MB';
-- ALTER SYSTEM SET work_mem = '128MB';
-- SELECT pg_reload_conf();
```

## 2. Schema 修改常見語法

```sql
-- 刪除欄位（多欄位一次刪除，安全語法 IF EXISTS）
ALTER TABLE 表名
DROP COLUMN IF EXISTS 欄位1,
DROP COLUMN IF EXISTS 欄位2;

-- 新增欄位
ALTER TABLE 表名 ADD COLUMN 欄位 VARCHAR(50);
ALTER TABLE 表名 ADD COLUMN 狀態 BOOLEAN DEFAULT FALSE;

-- 清空資料 + 重設序號（每次要成對執行）
DELETE FROM 表名;
ALTER SEQUENCE 表名_id_seq RESTART WITH 1;

-- 有外鍵關聯時，先清子表再清父表（依賴順序反過來刪）
DELETE FROM 子表2;
ALTER SEQUENCE 子表2_id_seq RESTART WITH 1;
DELETE FROM 子表1;
ALTER SEQUENCE 子表1_id_seq RESTART WITH 1;
DELETE FROM 父表;
ALTER SEQUENCE 父表_id_seq RESTART WITH 1;
```

## 3. Trigger 樣板：新增主表資料自動建立狀態子表

這是「主表 + 狀態子表」的常見設計 pattern，之後有類似需求可以直接套。

```sql
CREATE OR REPLACE FUNCTION insert_xxx_status()
RETURNS TRIGGER AS $$
BEGIN
	INSERT INTO XXX_STATUS (XXX_ID) VALUES (NEW.ID);
	RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_insert_xxx_status
AFTER INSERT ON XXX
FOR EACH ROW
EXECUTE FUNCTION insert_xxx_status();

-- update_at 自動更新（配合已存在的 update_updated_at_column() function）
CREATE TRIGGER update_xxx_at 
BEFORE UPDATE ON XXX 
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

主表常見索引組合（trgm 模糊搜尋 + GIST 空間 + 常用篩選欄位）：

```sql
CREATE INDEX IF NOT EXISTS idx_xxx_coord ON XXX USING GIST (COORD_GEOM);
CREATE INDEX IF NOT EXISTS idx_xxx_project_id ON XXX (PROJECT_ID);
CREATE INDEX IF NOT EXISTS idx_xxx_case_num_trgm ON XXX USING GIN (CASE_NUM gin_trgm_ops);
CREATE INDEX IF NOT EXISTS idx_xxx_address_trgm ON XXX USING GIN (ADDRESS gin_trgm_ops);
```

## 4. 兩來源資料比對範本（%匹配率）

用兩份清單依某個共同鍵值（例如地區）互相比對，估算涵蓋率／匹配率：

```sql
WITH total_roads AS (
    SELECT COUNT(DISTINCT ROAD1) AS total FROM 來源表A
),
matched_roads AS (
	SELECT COUNT(DISTINCT(A.ROAD1)) AS matched
	FROM 來源表A A
	JOIN 來源表B B ON A.REGION = B.REGION 
	WHERE (A.ROAD1 = B.ROAD OR A.ROAD2 = B.ROAD) AND B.RANKING > 0.65
)
SELECT (matched::FLOAT / total::FLOAT) * 100 AS match_percentage 
FROM total_roads, matched_roads;
```

## 5. 重複資料清除範本

```sql
BEGIN;
WITH DuplicateRecords AS (
  SELECT
    MIN(id) AS id_to_keep,
    ARRAY_AGG(id) AS all_ids
  FROM 表名
  GROUP BY DT_RECORD, IMG_DETECT, PRJ_ID, CRACKTYPE, DEGREE, CRACK_ID  -- 依實際判斷重複的欄位調整
  HAVING COUNT(*) > 1
)
DELETE FROM 表名
WHERE id = ANY (
  SELECT unnest(all_ids[2:])  -- 保留每組第一筆，刪除其餘
  FROM DuplicateRecords
);
COMMIT;
```

## 6. PostGIS 空間查詢技巧（OSM 路網：nodes / ways / way_nodes / way_tags）

```sql
-- 找路口（node 被 >1 條主要道路共用），排除人行道/機車道等
WITH intersections AS (
    SELECT wn.node_id, COUNT(DISTINCT wn.way_id) AS way_count, n.geom
    FROM way_nodes wn
    JOIN nodes n ON wn.node_id = n.id
	JOIN way_tags wt ON wn.way_id = wt.way_id
	WHERE wt.k = 'highway'
	 	AND wt.v NOT IN (
        	'sidewalk', 'crossing', 'footway', 'bridleway', 'pedestrian', 'cycleway',
        	'motorway', 'motorway_link', 'raceway', 'busway'
      	)
    GROUP BY wn.node_id, n.geom
    HAVING COUNT(DISTINCT wn.way_id) > 1
)
SELECT node_id, ST_Y(geom) AS lat, ST_X(geom) AS lon
FROM intersections
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(121.2187, 24.9633), 4326), 0.2);

-- 建成資料表 + 空間索引（大範圍路口資料先算好，之後查詢快很多）
CREATE TABLE INTERSECTIONS_TAIWAN AS (
  WITH intersections AS (
    SELECT wn.node_id, COUNT(DISTINCT wn.way_id) AS way_count, n.geom
    FROM way_nodes wn
    JOIN nodes n ON wn.node_id = n.id
    JOIN way_tags wt ON wn.way_id = wt.way_id
    WHERE wt.k = 'highway'
      AND wt.v NOT IN ('sidewalk','crossing','footway','bridleway','pedestrian','cycleway','motorway','motorway_link','raceway','busway')
    GROUP BY wn.node_id, n.geom
    HAVING COUNT(DISTINCT wn.way_id) > 1
  )
  SELECT node_id, geom FROM intersections
  WHERE ST_Within(geom, ST_MakeEnvelope(119.5, 21.8, 122.0, 25.4, 4326)) -- 台灣範圍
);
CREATE INDEX idx_intersections_taiwan_geom ON INTERSECTIONS_TAIWAN USING GIST(geom);

-- 找最近的一個路口 / 節點
SELECT node_id, ST_Y(geom) AS lat, ST_X(geom) AS lon,
    ST_Distance(geom, ST_SetSRID(ST_MakePoint(121.22042, 24.96262), 4326)) AS distance
FROM intersections_result
ORDER BY distance
LIMIT 1;

-- Prepared statement 版本（可重複帶入不同座標查詢）
PREPARE find_nearest_intersection(double precision, double precision, double precision) AS
SELECT node_id, ST_Y(geom) AS lat, ST_X(geom) AS lon
FROM intersections_result
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint($1, $2), 4326), $3)
ORDER BY geom <-> ST_SetSRID(ST_MakePoint($1, $2), 4326)
LIMIT 3;
-- EXECUTE find_nearest_intersection(121.195, 25.060, 0.0003);
-- DEALLOCATE find_nearest_intersection;

-- 依「方向角」挑選最符合前進方向的路口（轉彎判斷常用）
WITH current_and_next_point AS (
  SELECT
    ST_SetSRID(ST_MakePoint(121.195274, 25.05992), 4326) AS current_geom,
    ST_Azimuth(
      ST_SetSRID(ST_MakePoint(121.195274, 25.05992), 4326),
      ST_SetSRID(ST_MakePoint(121.195225, 25.059907), 4326)
    ) AS desired_direction
)
SELECT node_id, ST_Y(geom) AS lat, ST_X(geom) AS lon,
  ABS(ST_Azimuth(current_geom, geom) - desired_direction) AS direction_difference
FROM intersections_result, current_and_next_point
WHERE ST_DWithin(geom, current_geom, 0.0003)
ORDER BY direction_difference ASC, geom <-> current_geom
LIMIT 1;

-- 依地址 (city/district/street) 反查 OSM way，取路的中心點
SELECT ST_Centroid(ST_Collect(ND.geom)) AS center_geom
FROM way_nodes WN
JOIN nodes ND ON WN.node_id = ND.id
WHERE WN.way_id IN (
    SELECT wt1.way_id
    FROM way_tags AS wt1
    JOIN way_tags AS wt2 ON wt1.way_id = wt2.way_id
    WHERE wt1.k = 'addr:city' AND wt1.v = '臺中市'
      AND wt2.k = 'addr:district' AND wt2.v LIKE '%神岡區%'
);

-- 組合地址（city + district + street + housenumber）
SELECT DISTINCT ON (wt1.way_id)
  wt1.way_id,
  city.v AS city, district.v AS district, street.v AS street, housenumber.v AS housenumber,
  concat(street.v, COALESCE(housenumber.v, '')) AS full_address
FROM way_tags AS wt1
JOIN way_tags AS city ON wt1.way_id = city.way_id AND city.k = 'addr:city'
JOIN way_tags AS district ON wt1.way_id = district.way_id AND district.k = 'addr:district'
JOIN way_tags AS street ON wt1.way_id = street.way_id AND street.k = 'addr:street'
LEFT JOIN way_tags AS housenumber ON wt1.way_id = housenumber.way_id AND housenumber.k = 'addr:housenumber'
WHERE city.v = '桃園市' AND district.v = '八德區' AND street.v ILIKE '%長生路%'
ORDER BY wt1.way_id, housenumber.v NULLS LAST;
```

### 地址模糊比對（pg_trgm 相似度 + token 拆解）

用在「使用者輸入的地址字串」找最接近的正式地址，門檻可依需求調整 `similarity_threshold`：

```sql
BEGIN;
  SET LOCAL pg_trgm.similarity_threshold = 0.10;

WITH atxt AS (SELECT '仁愛路四段'::text AS term),
tokens AS MATERIALIZED (
  SELECT tok
  FROM (
    SELECT unnest(regexp_split_to_array(term, '(?<=(市|區|里|鄰|路|街|段|巷|弄|號|之))|[、,\s]+')) tok
    FROM atxt
  ) s
  WHERE length(tok) >= 2
  GROUP BY tok
  ORDER BY length(tok) DESC
  LIMIT 4
),
cand_prefix AS (
  SELECT id, n_address FROM address_lookup a, atxt
  WHERE lower(a.n_address) LIKE lower(atxt.term) || '%' LIMIT 50
),
cand_whole AS (
  SELECT id, n_address FROM address_lookup a CROSS JOIN atxt
  WHERE a.n_address % atxt.term
  ORDER BY a.n_address <-> atxt.term LIMIT 50
),
cand_tokens AS (
  SELECT DISTINCT a.id, a.n_address
  FROM tokens t
  JOIN LATERAL (
    SELECT id, n_address FROM address_lookup
	WHERE n_address % t.tok
	ORDER BY n_address <-> t.tok LIMIT 200
  ) a ON TRUE
),
candidates AS (
  SELECT DISTINCT id, n_address FROM (
    SELECT * FROM cand_prefix
    UNION ALL SELECT * FROM cand_whole
    UNION ALL SELECT * FROM cand_tokens
  ) u LIMIT 20
)
SELECT c.id, c.n_address,
  ((CASE WHEN lower(c.n_address) LIKE lower(atxt.term) || '%' THEN 1.0 ELSE 0.0 END)
    + 0.7 * similarity(c.n_address, atxt.term)
    + 0.2 * COALESCE(tok.avg_hit, 0)
    + 0.5 * COALESCE(tok.avg_wsim, 0)
  ) AS score
FROM candidates c
CROSS JOIN atxt
JOIN LATERAL (
  SELECT
    AVG(CASE WHEN c.n_address % t.tok THEN 1.0 ELSE 0.0 END) AS avg_hit,
    AVG(word_similarity(c.n_address, t.tok)) AS avg_wsim
  FROM tokens t
) AS tok ON TRUE
ORDER BY score DESC
LIMIT 5;
COMMIT;
```

## 7. 車輛軌跡分析（依時間分段成 LineString、依行政區統計）

判斷「連續軌跡點」何時應該斷成新的一段（起點標記 or 距離過大 or 行政區改變），再彙整成線段長度/時間統計：

```sql
WITH base AS (
	SELECT t.*, r.countyname, r.townname
  	FROM VEHICLE_TRACK t
	JOIN GIS_REGION r
  		ON r.countyname = '臺中市'
 		AND r.townname IN ('北區','北屯區')
		AND ST_Intersects(r.geom, ST_Transform(ST_SetSRID(ST_MakePoint(t.longitude, t.latitude), 4326), 3826))
  	WHERE t.DT_RECORD::DATE BETWEEN '2025-10-02' AND '2025-10-09'
      	AND t.PRJ_ID = 'TXG002' AND t.CAR = '3373-P5'
),
marked AS (
	SELECT b.*,
		SUM(CASE WHEN b.IS_START = '1' THEN 1 ELSE 0 END) OVER (ORDER BY b.DT_RECORD) AS segment_id,
		CASE
			WHEN b.IS_START = '1' THEN 1
			WHEN LAG(b.countyname) OVER (ORDER BY b.dt_record) IS DISTINCT FROM b.countyname THEN 1
			WHEN LAG(b.townname)   OVER (ORDER BY b.dt_record) IS DISTINCT FROM b.townname   THEN 1
			ELSE 0
		END AS region_break
	FROM base b
),
track_with_segment AS (
  SELECT m.*, SUM(m.region_break) OVER (ORDER BY m.dt_record) AS region_segment_id
  FROM marked m
),
points_with_dist AS (
  	SELECT *,
    	ST_SetSRID(ST_MakePoint(LONGITUDE, LATITUDE), 4326) AS geom,
    	LAG(ST_SetSRID(ST_MakePoint(LONGITUDE, LATITUDE), 4326)) OVER (PARTITION BY segment_id ORDER BY DT_RECORD) AS prev_point
  	FROM track_with_segment
),
filtered_points AS (
  	SELECT * FROM points_with_dist
  	WHERE prev_point IS NULL
    	OR ST_Distance(geom::geography, prev_point::geography) > 5   -- 濾掉幾乎沒動的雜訊點
),
seg AS (
 	SELECT region_segment_id, countyname, townname,
		ST_MakeLine(geom ORDER BY DT_RECORD) AS geom,
		MIN(DT_RECORD) AS start_time, MAX(DT_RECORD) AS end_time,
		(MAX(DT_RECORD) - MIN(DT_RECORD)) AS duration
	FROM filtered_points
	GROUP BY region_segment_id, countyname, townname
)
SELECT countyname, townname,
  COUNT(DISTINCT start_time::date) AS total_days,
  ROUND(SUM(ST_Length(geom::geography) / 1000.0)::numeric, 2) AS total_length_km,
  SUM(duration) AS total_duration
FROM seg
WHERE ST_Length(geom::geography) > 0
GROUP BY countyname, townname
ORDER BY countyname, townname;
```

小工具：把秒數轉成 `HH:MM:SS` 字串：

```sql
(
  (EXTRACT(EPOCH FROM 總時長)::bigint / 3600)::text || ':' ||
  lpad(((EXTRACT(EPOCH FROM 總時長)::bigint % 3600) / 60)::text, 2, '0') || ':' ||
  lpad((EXTRACT(EPOCH FROM 總時長)::bigint % 60)::text, 2, '0')
) AS time_hms
```

## 8. 中位數 / 分位數統計範本

```sql
-- 中位數
WITH Ordered AS (
    SELECT AREA,
           ROW_NUMBER() OVER (ORDER BY AREA ASC) AS row_num,
           COUNT(*) OVER () AS total_rows
    FROM 表名
    WHERE 篩選條件
)
SELECT AVG(AREA) AS median_area
FROM Ordered
WHERE row_num IN ((total_rows + 1) / 2, (total_rows + 2) / 2);

-- 任意分位數（例：Q3 = 75%）
WITH Ordered AS (
    SELECT AREA,
           ROW_NUMBER() OVER (ORDER BY AREA ASC) AS row_num,
           COUNT(*) OVER () AS total_rows
    FROM 表名
    WHERE 篩選條件
)
SELECT AREA AS q3_area
FROM Ordered
WHERE row_num = CEIL(total_rows * 0.75);
```

## 9. Docker 部署指令備忘

```bash
# 禪道 (ZenTao) 專案管理系統
docker run -d \
    -v $PWD/data:/data \
    -p 666:80 \
    -e MYSQL_INTERNAL=true \
    hub.zentao.net/app/zentao:latest
```

---

## 附錄：危險操作模式（僅供理解 pattern，實際使用前務必確認環境與備份）

以下是原始筆記中常出現的「清空重建」操作寫法，記錄下來是為了保留 **模式**（哪些表要一起清、序號怎麼重設），
**不要不假思索地照抄執行**：

```sql
-- 模式一：整批清空某功能相關的表（含子表與序號）
DELETE FROM 子表2;
DELETE FROM 子表1;
DELETE FROM 主表;

-- 模式二：條件式清除（保留某個時間點以前的資料）
DELETE FROM 表A WHERE CREATED_AT > '指定時間';
DELETE FROM 表B WHERE CREATED_AT > '指定時間';
DELETE FROM 表C WHERE CREATED_AT > '指定時間';

-- 模式三：把測試/髒資料的時間欄位重打（例如把日期補成當天，保留原本時間部分）
UPDATE 表名 SET DT_RECORD = CURRENT_DATE + DT_RECORD::time;
```
