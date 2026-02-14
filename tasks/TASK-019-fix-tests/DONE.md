# TASK-019 完成報告

## 修改內容

### 1. test-source-mgmt.sh — 修復
- 新增 `adminUsername` 和 `adminPassword` 參數到 create API
- 新增清理殘留測試資料邏輯（防 UNIQUE 衝突）

### 2. test-device-mgmt.sh — 修復
- 同上，修復站台建立時缺少 adminUsername/adminPassword
- 新增清理殘留邏輯

### 3. test-auto-order.sh — 補充 2 項測試
- 代收訂單門市地址驗證（receiverProvince + 7-11）
- 代付單筆退貨驗證（退貨數量=1, returnAmount=200）

### 4. test-export.sh — 新增 5 項測試
- 訂單導出無條件 → 400
- 訂單導出有條件 → 200 + xlsx
- 退貨導出無條件 → 400
- 退貨導出有條件 → 200 + xlsx
- 日期超過31天 → 400

### 5. test-store711.sh — 新增 3 項測試
- 7-11 門市總數 >6000
- 縣市覆蓋 >=22
- 假人全部綁定門市

### 6. 額外修復
- test-store711.sh 中 `TOTAL` 變數名與 run-tests.sh 全域計數器衝突，改名為 `STORE_TOTAL`

## 最終測試結果

```
🧪 General Mall API Tests
  Admin: http://127.0.0.1:8080
  Portal: http://127.0.0.1:8085
================================

📋 test-auth: 後台登入
  ✅ PASS: 正確帳密登入 → 200 + token
  ✅ PASS: 錯誤密碼 → 非 200 (code=500)

📋 test-source-mgmt: 來源管理
  ✅ PASS: 來源列表 → 200, 有 3 筆資料
  ✅ PASS: 建立來源 → 200, id=22, 帳號=test_auto_api, 有密碼和 apiToken
  ✅ PASS: 查詢來源 → 200, name=test_auto_api
  ✅ PASS: 重新產生 token → 200, token 已更新
  ✅ PASS: 更新來源狀態 → 200
  ✅ PASS: 刪除來源 → 200 (清理完成)

📋 test-device-mgmt: 設備管理
  ✅ PASS: 建立設備 → 200
  ✅ PASS: 設備列表 → 200, 包含剛建立的設備
  ✅ PASS: 更新設備 → 200
  ✅ PASS: 刪除設備 → 200

📋 test-auto-order: 代收代付
  ✅ PASS: 無 token → 非 200 (code=500)
  ✅ PASS: 錯誤 token → 非 200 (code=500)
  ✅ PASS: 錯誤 employeeId → 非 200 (code=500)
  ✅ PASS: 錯誤 deviceId → 非 200 (code=500)
  ✅ PASS: 代收 → 200, orderId=202602131630356632
  ✅ PASS: orderId 格式正確 (18位數字)
  ✅ PASS: 代收 amount=0 → 失敗 (code=404)
  ✅ PASS: 缺少 employeeId → 失敗 (code=404)
  ✅ PASS: 代付 → 200, returnOrderId=R202602131630357936 (R開頭)
  ✅ PASS: 代付 amount=0 → 失敗 (code=404)
  ✅ PASS: 訂單驗證 → receiverName 含中文, phone 09 開頭
  ✅ PASS: 代收訂單帶入門市地址 → 南投縣, 含7-11
  ✅ PASS: 代付只建一筆退貨申請 (orderSn=R202602131630357936)
  ✅ PASS: 代付 returnAmount = 200

📋 test-order-admin: 後台訂單管理
  ✅ PASS: 訂單列表 → 200, 共 10 筆
  ✅ PASS: 修改 receiverInfo → 200
  ✅ PASS: 退貨申請列表 → 200
  ✅ PASS: 訂單詳情 → 200
  ✅ PASS: 更新訂單備註 → 200
  ✅ PASS: 退貨原因列表 → 200

📋 test-brand: 品牌管理
  ✅ PASS: 品牌列表 → 200
  ✅ PASS: 品牌全部列表 → 200
  ✅ PASS: 建立品牌 → 200
  ✅ PASS: 更新品牌 → 200
  ✅ PASS: 刪除品牌 → 200 (清理完成)

📋 test-coupon: 優惠券管理
  ✅ PASS: 優惠券列表 → 200
  ✅ PASS: 建立優惠券 → 200
  ✅ PASS: 查詢優惠券 → 200
  ✅ PASS: 更新優惠券 → 200
  ✅ PASS: 刪除優惠券 → 200 (清理完成)

📋 test-export: 導出 Excel
  ✅ PASS: 訂單導出無條件 → 400 拒絕
  ✅ PASS: 訂單導出有條件 → 200, 檔案 4899 bytes
  ✅ PASS: 退貨導出無條件 → 400 拒絕
  ✅ PASS: 退貨導出有條件 → 200, 檔案 5760 bytes
  ✅ PASS: 訂單導出超過31天 → 400 拒絕

📋 test-home-content: 首頁內容管理
  ✅ PASS: 廣告列表 → 200
  ✅ PASS: 首頁品牌列表 → 200
  ✅ PASS: 新品推薦列表 → 200
  ✅ PASS: 人氣推薦列表 → 200

📋 test-portal: 前台 Portal API
  ✅ PASS: Portal 首頁內容 → 200
  ✅ PASS: Portal 分類樹 → 200
  ✅ PASS: Portal 商品搜尋 → 200
  ✅ PASS: Portal 推薦品牌 → 200

📋 test-product-admin: 商品管理
  ✅ PASS: 商品列表 → 200, total=560
  ✅ PASS: 商品簡單列表 → 200
  ✅ PASS: 建立測試商品 → 200
  ✅ PASS: 查詢商品詳情 → 200
  ✅ PASS: 更新商品 → 200
  ✅ PASS: 上架商品 → 200
  ✅ PASS: 刪除商品 → 200 (清理完成)

📋 test-product-category: 商品分類
  ✅ PASS: 分類列表(parentId=0) → 200
  ✅ PASS: 分類樹狀列表 → 200
  ✅ PASS: 建立分類 → 200
  ✅ PASS: 更新分類 → 200
  ✅ PASS: 刪除分類 → 200 (清理完成)

📋 test-store711: 7-11 門市資料
  ✅ PASS: 7-11 門市總數 = 7316 (>6000)
  ✅ PASS: 7-11 涵蓋 22 個縣市 (>=22)
  ✅ PASS: 全部假人都有綁定門市 (10000/10000)

📋 test-ums: 用戶/角色/菜單/資源
  ✅ PASS: 管理員列表 → 200
  ✅ PASS: 管理員資訊 → 200, username=admin
  ✅ PASS: 角色列表 → 200
  ✅ PASS: 角色全部列表 → 200
  ✅ PASS: 菜單樹狀列表 → 200
  ✅ PASS: 資源列表 → 200
  ✅ PASS: 資源全部列表 → 200
  ✅ PASS: 資源分類列表 → 200

================================
Total: 78 | Pass: 78 | Fail: 0
================================
```

## 驗收
- ✅ 全部 78 測試 PASS（0 FAIL）
- ✅ 測試數量 78 >= 75
- ✅ 覆蓋：站台建立、代收門市地址、代付單筆退貨、導出 Excel、7-11 資料
- ✅ 未修改任何後端/前端程式碼
