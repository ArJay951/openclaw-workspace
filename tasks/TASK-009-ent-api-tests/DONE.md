# TASK-009 DONE

## 結果：✅ 全部 PASS（23/23）

## 產出檔案
- `/home/ubuntu/entertainment-mall/tests/run-tests.sh` — 主執行腳本
- `/home/ubuntu/entertainment-mall/tests/test-auth.sh` — 登入測試（3 cases）
- `/home/ubuntu/entertainment-mall/tests/test-tenant.sh` — 多租戶測試（5 cases）
- `/home/ubuntu/entertainment-mall/tests/test-home.sh` — 首頁 API（2 cases）
- `/home/ubuntu/entertainment-mall/tests/test-product.sh` — 商品 API（3 cases）
- `/home/ubuntu/entertainment-mall/tests/test-security.sh` — 資安驗證（5 cases）
- `/home/ubuntu/entertainment-mall/tests/test-admin-crud.sh` — 後台 CRUD（5 cases）

## 測試覆蓋
- Auth: admin 正確/錯誤登入, portal 端點確認
- Tenant: 正確域名回傳 brandName/themeColor, 無 tenantCode, fake domain 403, localhost fallback 200, admin tenant/info
- Home: /home/content, /product/categoryTreeList
- Product: portal search, portal detail, admin list (含資料量驗證)
- Security: tenant/refresh portal 404, admin 無 token 401, actuator nginx 403 (portal+admin), tenantCode 深層檢查
- Admin CRUD: brand/list, productCategory/list, order/list, returnApply/list, coupon/list

## 執行方式
```bash
cd /home/ubuntu/entertainment-mall/tests && ./run-tests.sh
```
