# TASK-005: DONE ✅

## 完成時間
2026-02-12 14:30 UTC

## 完成項目

### 1. 建表 + 塞資料 ✅
- `oms_fake_contact` 表已存在，含 10,000 筆資料
- 姓名：2~3 字台灣中文名（如：施貞、何鳳、呂彬妤）
- 手機：09 開頭 10 位數字

### 2. Model + Mapper ✅
- `OmsFakeContact.java` — id, name, phone
- `OmsFakeContactMapper.java` — `selectRandom()`（高效隨機查詢）、`selectCount()`
- 使用 annotation-based mapper（@Select）

### 3. AutoOrderServiceImpl 修改 ✅
- `generateCollectOrder()`: receiverName/receiverPhone 用假人資料
- `generatePayOrder()`: receiverName/receiverPhone + returnName/returnPhone 都用假人資料
- fallback：若假人表為空，用 employeeId / 0900000000

### 4. 後台修改功能 ✅
- 訂單：`POST /order/update/receiverInfo` — 可修改 receiverName, receiverPhone
- 退貨申請：`POST /returnApply/update/returnInfo` — 可修改 returnName, returnPhone

### 5. Build ✅
- `mvn clean package -DskipTests` — BUILD SUCCESS

### 6. 測試結果 ✅

**代收測試:**
```
POST /order/auto-generate/collect (amount=500)
→ 訂單 202602121429186275, receiverName=施貞, receiverPhone=0986895720
```

**代付測試:**
```
POST /order/auto-generate/pay (amount=300)
→ 訂單 R202602121429280023, receiverName=何鳳, receiverPhone=0996443244
→ 退貨申請 returnName=何鳳, returnPhone=0996443244
```

**後台修改測試:**
```
POST /order/update/receiverInfo → 200 OK
POST /returnApply/update/returnInfo → 200 OK
```

## 備註
- Docker 容器已重啟（mall-portal, mall-admin）使用新 jar
- API Token header 名稱是 `X-Api-Token`（非 `X-Auto-Order-Token`）
- Admin 密碼: `123456`
