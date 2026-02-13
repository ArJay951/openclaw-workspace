# TASK-011: 代付只建退貨申請，不建假訂單

## 目標
修改代付 API (`/order/auto-generate/pay`)，只建立 `oms_order_return_apply`，不再建立 `oms_order` 和 `oms_order_item`。

## 修改檔案
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java` — `generatePayOrder()` 方法

## 具體改動

### generatePayOrder() 改造：
1. **保留**：商品選擇算法（selectProductsMultiRound）— 用來填退貨申請的商品資訊
2. **保留**：建立 `oms_order_return_apply`
3. **刪除**：建立 `oms_order` 和 `oms_order_item` 的所有程式碼
4. **調整**：退貨申請的欄位
   - `orderId` → 設為 0 或 null（沒有對應訂單了）
   - `orderSn` → 用生成的 returnOrderId（R + 時間戳）
   - `returnAmount` → request.getAmount()
   - 其他欄位保持不變（returnName/Phone 用假人、商品資訊用選出的第一個商品）

### AutoOrderResult 調整：
- `paySuccess()` 回傳的結果，移除跟 order 相關的欄位（如果有的話）
- 確保回傳 returnOrderId + 金額資訊

## 驗收標準
1. `mvn clean package -DskipTests` 成功
2. Docker 容器重啟成功
3. curl 代付 API → 200，回傳 returnOrderId
4. DB 確認：只有 `oms_order_return_apply` 新增一筆，`oms_order` 沒有新增
5. 代收 API 不受影響（還是正常建 order）
6. 跑 `/home/ubuntu/general-mall/tests/run-tests.sh`，修正受影響的測試，全部 PASS
