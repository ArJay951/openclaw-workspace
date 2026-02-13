# TASK-016 完成報告

## 狀態：✅ 完成

## 完成項目

### 後端
1. **OmsOrderQueryParam** - `createTime` 改為 `startTime` + `endTime` 日期區間
2. **OmsOrderDao.xml** - 查詢條件從 LIKE 改為 `>=` / `<=` 區間查詢
3. **OmsOrderService** - 新增 `listAll()` 方法（不用 PageHelper）
4. **OmsOrderController** - 新增 `/order/exportExcel` GET API
   - 驗證至少一個篩選條件
   - 日期區間不超過 31 天
   - 使用 hutool ExcelWriter 輸出 xlsx
   - 包含欄位：訂單編號、提交時間、收貨人、手機號碼、訂單金額、應付金額、站台名稱、訂單狀態、收貨地址、商品明細
   - 商品明細格式：`商品A × 2\n商品B × 1`
   - 站台帳號 sourceFilter 自動套用
5. **poi-ooxml 4.1.2** 依賴加入 mall-admin（與現有 commons-compress 相容）

### 前端
1. **提交時間** - 改為 `el-date-picker type="daterange"` 日期區間選擇器
2. **導出按鈕** - 在搜索按鈕旁新增「導出Excel」按鈕
3. **request.js** - response interceptor 加入 blob responseType 判斷，直接返回 blob data
4. **api/order.js** - 新增 `exportExcel()` 函數（axios blob 方式下載）
5. **handleExport()** - 前端驗證 + blob 下載 + 自動觸發檔案下載

## 驗收結果
- ✅ `mvn clean package -DskipTests` 成功
- ✅ `npm run build` 成功
- ✅ Docker 部署成功
- ✅ API 測試：`/order/exportExcel?startTime=2026-02-01&endTime=2026-02-28` 回傳 4873 bytes xlsx 檔案
- ✅ `file` 命令確認為 `Microsoft Excel 2007+`

## 踩坑記錄
- poi-ooxml 5.2.5 與專案現有 commons-compress 版本衝突（`NoSuchMethodError`），降版至 4.1.2 解決

## Git Commits
- 後端: `feat: 訂單列表時間區間篩選+導出Excel（含商品明細）` + `fix: poi-ooxml 降版至4.1.2修復commons-compress版本衝突`
- 前端: `feat: 訂單列表時間區間篩選+導出Excel`
