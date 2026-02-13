# TASK-009: Entertainment Mall API 自動測試腳本

## 目標
建立一組 shell 測試腳本，覆蓋 Entertainment Mall 的核心 API，部署後一鍵驗證。

## 產出
- `/home/ubuntu/entertainment-mall/tests/run-tests.sh` — 主執行腳本
- `/home/ubuntu/entertainment-mall/tests/test-*.sh` — 各模組測試

---

## 測試覆蓋範圍

### 1. test-auth.sh（後台 + 前台登入）
Admin (port 8090):
- ✅ 正確帳密登入 → 200 + token
- ❌ 錯誤密碼 → 非 200

Portal (port 8095):
- ✅ 會員註冊或登入 → 200 + token（如果有公開註冊）
- 或至少確認登入端點回應正常

### 2. test-tenant.sh（多租戶功能）
Portal:
- ✅ GET /tenant/info（Host: ent.homely-go.com）→ 200，回傳 brandName, themeColor, brandLogo
- ✅ 確認不回傳 tenantCode
- ❌ GET /tenant/info（Host: fake.example.com）→ 403 Unknown tenant
- ❌ GET /tenant/info（Host: localhost）→ 200（localhost fallback 正常）

Admin:
- ✅ GET /tenant/info → 200

### 3. test-home.sh（前台首頁 API）
Portal (port 8095):
- ✅ GET /home/content → 200（首頁內容）
- ✅ GET /product/categoryTreeList → 200（分類樹）

### 4. test-product.sh（商品 API）
Portal:
- ✅ GET /product/search?keyword=test → 200
- ✅ GET /product/detail/{id} → 200（用一個已知商品 ID）

Admin:
- ✅ GET /product/list → 200（需 JWT）
- ✅ 確認商品列表有資料

### 5. test-security.sh（資安驗證）
- ❌ POST /tenant/refresh（Portal）→ 404（已移除）
- ❌ POST /tenant/refresh（Admin 無 token）→ 401
- ❌ GET /actuator（Portal）→ 403（被 Nginx 擋）
- ❌ GET /actuator（Admin）→ 403（被 Nginx 擋）
- ✅ 確認 tenant/info 不含 tenantCode

### 6. test-admin-crud.sh（後台 CRUD）
需要 admin JWT

- ✅ GET /brand/list → 200
- ✅ GET /productCategory/list/0 → 200（分類列表）
- ✅ GET /order/list → 200（訂單列表）
- ✅ GET /returnApply/list → 200（退貨列表）
- ✅ GET /coupon/list → 200（優惠券列表）

---

## 腳本規範

與 TASK-008（General Mall）相同格式：
- `run-tests.sh` 主腳本，支援 `./run-tests.sh [base_url]`，預設 `http://127.0.0.1`
- PASS/FAIL 輸出 + 最終統計
- 用 curl + jq
- 不建立測試資料（只讀測試），避免污染娛樂城 DB
- 確認 `jq` 已安裝

---

## 驗收標準
1. `./run-tests.sh` 一鍵執行，全部 PASS
2. 覆蓋至少 20 個測試 case
3. 多租戶 + 資安測試是重點
4. 腳本有註解，易讀
