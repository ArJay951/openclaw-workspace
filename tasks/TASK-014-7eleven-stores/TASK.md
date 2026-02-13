# TASK-014: 爬取全台 7-11 門市資料

## 目標
爬取全台灣所有 7-11 門市資料，產生可攜式 SQL 檔。

## 資料來源
emap.pcsc.com.tw 的 EMapSDK API

### API 說明
- URL: `https://emap.pcsc.com.tw/EMapSDK.aspx` (POST)
- 需要帶 cookie: 先打一次 GetTown 取得 session cookie，後續請求帶上

### 取得區域
```
commandid=GetTown&cityid={cityid}
```
回傳 XML: `<GeoPosition><TownID>01</TownID><TownName>松山區</TownName>...</GeoPosition>`

### 搜尋門市
```
commandid=SearchStore&city={城市名}&town={區域名}
```
回傳 XML，每個 `<GeoPosition>` 包含:
- `POIID`: 門市代碼（trim 空白）
- `POIName`: 門市名稱
- `Address`: 地址
- `Telno`: 電話（trim 空白）
- `FaxNo`: 傳真
- `X`: 經度（需除以 1000000 或處理小數點格式）
- `Y`: 緯度（同上）
- `StoreImageTitle`: 服務項目（逗號分隔）
- `OP_DAY`: 營業日備註
- `OP_TIME`: 營業時間

### 縣市代碼對照表
```
01=台北市, 02=基隆市, 03=新北市, 04=桃園市, 05=新竹市,
06=新竹縣, 07=苗栗縣, 08=台中市, 09=彰化縣, 10=南投縣,
11=雲林縣, 12=嘉義市, 13=嘉義縣, 14=台南市, 15=高雄市,
16=屏東縣, 17=宜蘭縣, 18=花蓮縣, 19=台東縣, 20=澎湖縣,
21=金門縣, 22=連江縣
```

## 步驟

### 1. 寫爬取腳本 (Python)
路徑: `/home/ubuntu/7eleven-scraper/scrape.py`

```python
# 邏輯:
# 1. 對每個 cityid 呼叫 GetTown 取區域列表
# 2. 對每個 city+town 呼叫 SearchStore 取門市
# 3. 解析 XML，trim 所有欄位
# 4. 去重（同一個 POIID 只保留一筆）
# 5. 存成 JSON: /home/ubuntu/7eleven-scraper/stores.json
# 6. 印統計: 每個城市的門市數量
```

注意：
- 每次請求間隔 0.5 秒，避免被封
- 經緯度處理：有的是整數（如 121577218），有的帶小數（如 121548287.390895），統一轉成標準小數格式（121.577218）
- POIID、Telno、FaxNo 需要 trim 空白
- 若某區域回傳空結果，跳過不報錯
- 使用 requests + session 保持 cookie

### 2. 產生 SQL
路徑: `/home/ubuntu/7eleven-scraper/generate_sql.py`

產出: `/home/ubuntu/7eleven-scraper/7eleven_stores.sql`

```sql
-- 建表
DROP TABLE IF EXISTS `store_711`;
CREATE TABLE `store_711` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `store_id` varchar(20) NOT NULL COMMENT '門市代碼 (POIID)',
  `store_name` varchar(50) NOT NULL COMMENT '門市名稱',
  `city` varchar(10) NOT NULL COMMENT '縣市',
  `district` varchar(10) NOT NULL COMMENT '區域',
  `address` varchar(200) NOT NULL COMMENT '地址',
  `phone` varchar(20) DEFAULT NULL COMMENT '電話',
  `longitude` decimal(10,6) DEFAULT NULL COMMENT '經度',
  `latitude` decimal(10,6) DEFAULT NULL COMMENT '緯度',
  `services` text COMMENT '服務項目',
  `op_day` varchar(100) DEFAULT NULL COMMENT '營業日備註',
  `op_time` varchar(50) DEFAULT NULL COMMENT '營業時間',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_store_id` (`store_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='7-11門市資料';

-- INSERT 資料（每 100 筆一個 INSERT）
INSERT INTO `store_711` (store_id, store_name, city, district, address, phone, longitude, latitude, services, op_day, op_time) VALUES
(...),
(...);
```

### 3. 匯入 MySQL（開發環境）
```bash
docker exec -i mall-mysql mysql -u root -proot mall < /home/ubuntu/7eleven-scraper/7eleven_stores.sql
```

## 驗收標準
1. 爬到 6000+ 筆門市（全台約 6,800+ 家）
2. SQL 匯入成功，`SELECT COUNT(*) FROM store_711` 驗證
3. 每個縣市都有資料
4. 經緯度格式正確（台灣範圍: 經度 119~122, 緯度 21~26）
5. stores.json 備份存在

## ⚠️ 注意
- 請求間隔 >= 0.5 秒
- 若被限流（回傳空或錯誤），等 5 秒重試，最多重試 3 次
- SQL 檔用 utf8mb4
- 只操作**開發環境** DB
