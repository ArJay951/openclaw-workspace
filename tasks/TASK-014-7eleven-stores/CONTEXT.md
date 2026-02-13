# CONTEXT - TASK-014

## 路徑
- 工作目錄: `/home/ubuntu/7eleven-scraper/`
- 輸出 SQL: `/home/ubuntu/7eleven-scraper/7eleven_stores.sql`
- 輸出 JSON: `/home/ubuntu/7eleven-scraper/stores.json`

## 開發環境 DB
- MySQL: `docker exec -i mall-mysql mysql -u root -proot mall`
- DB: `mall`

## 已驗證的 API 呼叫

### GetTown（成功）
```bash
curl -s -X POST 'https://emap.pcsc.com.tw/EMapSDK.aspx' -d 'commandid=GetTown&cityid=01'
```
回傳台北市 12 個區。

### SearchStore（成功）
```bash
curl -s -X POST 'https://emap.pcsc.com.tw/EMapSDK.aspx' -d 'commandid=SearchStore&city=台北市&town=松山區'
```
回傳松山區約 60 家門市。

### GetCity（失敗，回傳空）
不要用這個 API，改用硬編碼的縣市代碼。

## 經緯度格式
API 回傳兩種格式：
- 整數: `121577218` → 轉成 `121.577218`
- 帶小數: `121548287.390895` → 取整數部分 `121548287` 再轉 `121.548287`

統一處理邏輯：
```python
def parse_coord(val):
    # 移除小數點後的部分（如果有），取整數，再除以 1000000
    parts = val.split('.')
    integer_part = int(parts[0])
    if integer_part > 1000000:
        return integer_part / 1000000.0
    return float(val)
```

## Python 依賴
- requests（已安裝）
- xml.etree.ElementTree（內建）

## 統計參考
- 全台 7-11 約 6,800+ 家（2024 年資料）
- 台北市約 700+ 家
- 新北市約 800+ 家
