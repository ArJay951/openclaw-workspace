# TASK-015 完成報告

## 完成項目

### 1. DB 變更
- ✅ `oms_fake_contact` 新增 `store_id` 欄位 (VARCHAR(20))
- ✅ `oms_order_return_apply.product_attr` 從 VARCHAR(500) 改為 TEXT（修正既有 bug）

### 2. 門市分配
- ✅ 10000 假人全部分配 store_id（100%）
- ✅ 按台灣人口比例分配，各縣市分布：
  - 新北市: 1703, 台中市: 1202, 高雄市: 1182, 台北市: 1082, 桃園市: 982
  - 台南市: 802, 彰化縣: 531, 屏東縣: 341 ... 連江縣: 13
- ✅ 腳本: `/home/ubuntu/7eleven-scraper/assign_stores.py`

### 3. Java 程式碼
- ✅ `OmsFakeContact.java` — 新增 storeId + 5 個 transient 門市欄位
- ✅ `OmsFakeContactMapper.java` — 新增 `selectRandomWithStore()` (LEFT JOIN store_711)
- ✅ `AutoOrderServiceImpl.java` — 代收用 selectRandomWithStore，帶入 receiverProvince/City/DetailAddress
- ✅ `AutoOrderServiceImpl.java` — 代付用 selectRandomWithStore

### 4. Build & Deploy
- ✅ `mvn clean package -DskipTests` 成功
- ✅ Docker build mall-admin + mall-portal
- ✅ Docker compose up 成功

### 5. 測試結果
- ✅ 代收訂單帶入門市地址：`台中市 / 北區 / 7-11 中友 (台中市北區育才北路74號76號)`
- ✅ 代付退貨正常運作

### 6. Git
- ✅ Commit: `feat: 假人資料關聯7-11門市，代收訂單自動帶入門市地址`
- ✅ Push to GitHub

## 額外修正
- `oms_order_return_apply.product_attr` 欄位太小 (VARCHAR 500)，多商品 JSON 會超出，已改為 TEXT

## 測試用 API
```bash
# 代收（用 claw source token）
curl -s -X POST http://localhost:8085/order/auto-generate/collect \
  -H "Content-Type: application/json" \
  -H "X-Api-Token: 36ae430b6120992e5ac779cd8713342e" \
  -d '{"employeeId":"emp001","deviceId":"dev001","amount":300}'

# 代付
curl -s -X POST http://localhost:8085/order/auto-generate/pay \
  -H "Content-Type: application/json" \
  -H "X-Api-Token: 36ae430b6120992e5ac779cd8713342e" \
  -d '{"employeeId":"emp001","deviceId":"dev001","amount":500}'
```

注意：TASK.md 裡的測試 token `550711ffb944efb498366a45940f736e` (本地測試) 沒有綁定 device，需用 claw source 的 token 測試。
