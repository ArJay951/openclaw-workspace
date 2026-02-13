# TASK-018 完成報告

## 狀態：✅ 完成

## 變更摘要

### 後端（mall-backend）
1. **OmsReturnApplyQueryParam.java** — `createTime` 改為 `startTime` / `endTime` 日期區間
2. **OmsOrderReturnApplyDao.xml** — createTime LIKE 改為日期區間查詢；SELECT 加入 product_name、product_attr
3. **OmsOrderReturnApplyService.java** — 新增 `listAll()` 不分頁查詢方法
4. **OmsOrderReturnApplyServiceImpl.java** — 實作 `listAll()`
5. **OmsOrderReturnApplyController.java** — 新增 `exportExcel` API，含：
   - 至少一個篩選條件驗證
   - 日期區間≤31天驗證
   - 站台帳號過濾（applySourceFilter）
   - Excel 欄位：服務單號、訂單編號、申請時間、退貨人、手機號碼、退款金額、申請狀態、退貨原因、問題描述、商品明細
   - 商品明細：代付訂單(orderId=0)從 productAttr JSON 解析，一般退貨用 productName × productCount

### 前端（mall-admin-web）
1. **returnApply.js** — 新增 `exportExcel` API（responseType: blob）
2. **apply/index.vue** — 
   - 申請時間改為 daterange 日期區間選擇器
   - 新增「導出Excel」按鈕（warning 樣式）
   - handleExport：條件驗證 + 31天限制 + blob 下載
   - getList 改用 buildParams() 處理 dateRange → startTime/endTime

## 驗收結果
- ✅ mvn clean package -DskipTests 成功
- ✅ npm run build 成功
- ✅ Docker 重建 + 部署完成
- ✅ Git push 完成（後端 0296a7c、前端 778bd8b）
