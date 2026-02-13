# TASK-010: 補齊系統原有邏輯測試

## 目標
在現有測試腳本基礎上，補齊 macrozheng/mall 框架原有功能的 API 測試。
兩個商城都要補。

## ⚠️ 原則
- 只讀測試為主（GET），CRUD 測試要自動清理
- 不破壞現有資料
- 追加到現有 tests/ 目錄，新增 test-*.sh 檔案
- run-tests.sh 要自動包含新增的測試

---

## General Mall (`/home/ubuntu/general-mall/tests/`)

### test-product-admin.sh（商品管理）
- ✅ GET /product/list → 200，有資料
- ✅ GET /product/simpleList → 200
- ✅ POST /product/create → 200（建立測試商品）
- ✅ GET /product/updateInfo/{id} → 200（查剛建的）
- ✅ POST /product/update/{id} → 200（更新）
- ✅ POST /product/update/publishStatus → 200（上下架）
- ✅ POST /product/update/deleteStatus → 200（刪除，清理）

### test-product-category.sh（商品分類）
- ✅ GET /productCategory/list/0 → 200
- ✅ GET /productCategory/list/withChildren → 200
- ✅ POST /productCategory/create → 200
- ✅ POST /productCategory/update/{id} → 200
- ✅ POST /productCategory/delete/{id} → 200（清理）

### test-brand.sh（品牌管理）
- ✅ GET /brand/list → 200
- ✅ GET /brand/listAll → 200
- ✅ POST /brand/create → 200
- ✅ POST /brand/update/{id} → 200
- ✅ POST /brand/delete/{id} → 200（清理）

### test-order-admin.sh（擴充現有）
- ✅ GET /order/list → 200
- ✅ GET /order/{id} → 200（用已知訂單）
- ✅ POST /order/update/note → 200
- ✅ GET /returnApply/list → 200
- ✅ GET /returnReason/list → 200

### test-coupon.sh（優惠券）
- ✅ GET /coupon/list → 200
- ✅ POST /coupon/create → 200
- ✅ GET /coupon/{id} → 200
- ✅ POST /coupon/update/{id} → 200
- ✅ POST /coupon/delete/{id} → 200（清理）

### test-home-content.sh（首頁內容管理）
- ✅ GET /home/advertise/list → 200
- ✅ GET /home/brand/list → 200
- ✅ GET /home/newProduct/list → 200
- ✅ GET /home/recommendProduct/list → 200

### test-ums.sh（用戶/角色/菜單/資源）
- ✅ GET /admin/list → 200
- ✅ GET /admin/info → 200（用 JWT）
- ✅ GET /role/list → 200
- ✅ GET /role/listAll → 200
- ✅ GET /menu/treeList → 200
- ✅ GET /resource/list → 200
- ✅ GET /resource/listAll → 200
- ✅ GET /resourceCategory/listAll → 200

### test-portal.sh（前台 Portal API，port 8085）
- ✅ GET /home/content → 200（首頁內容）
- ✅ GET /product/categoryTreeList → 200
- ✅ GET /product/search?keyword=test → 200
- ✅ GET /brand/recommendList → 200

---

## Entertainment Mall (`/home/ubuntu/entertainment-mall/tests/`)

### test-product-admin.sh（商品管理）
- ✅ GET /product/list → 200
- ✅ POST /product/create → 200
- ✅ POST /product/update/deleteStatus → 200（清理）

### test-product-category.sh（商品分類）
- ✅ GET /productCategory/list/0 → 200
- ✅ GET /productCategory/list/withChildren → 200

### test-brand.sh（品牌管理）
- ✅ GET /brand/list → 200
- ✅ GET /brand/listAll → 200

### test-coupon.sh（優惠券）
- ✅ GET /coupon/list → 200

### test-ums.sh（用戶/角色/菜單）
- ✅ GET /admin/list → 200
- ✅ GET /admin/info → 200
- ✅ GET /role/list → 200
- ✅ GET /menu/treeList → 200

### test-portal-full.sh（前台 Portal，port 8095）
需帶 Host: ent.homely-go.com

- ✅ GET /home/content → 200
- ✅ GET /product/categoryTreeList → 200
- ✅ GET /product/search?keyword=test → 200
- ✅ GET /brand/recommendList → 200 或合理回應

---

## 腳本規範
- 與現有格式一致（pass/fail 函數、統計）
- 新 test-*.sh 都要被 run-tests.sh 自動載入
- 確認 run-tests.sh 用 `for f in test-*.sh` 或類似方式動態載入（如果不是，改成動態載入）
- CRUD 測試建立的資料一定要在最後清理

## 驗收標準
1. 兩個商城 run-tests.sh 全部 PASS
2. General Mall 總測試數 ≥ 60
3. Entertainment Mall 總測試數 ≥ 40
4. 無測試資料殘留
