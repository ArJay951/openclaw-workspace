# TASK-002: SaaS 資安修復 — 完成報告

## 完成時間: 2026-02-12 13:07 UTC

## 修復結果

### ✅ Fix 1: /tenant/refresh 加認證
- Portal: 移除了 `/tenant/refresh` 端點
- Admin: 從安全白名單移除 `/tenant/**`，改為只放行 `/tenant/info`
- 驗證: Portal POST /tenant/refresh → 404 ✅, Admin 無 token → 401 ✅

### ✅ Fix 2: 關閉 domain fallback
- TenantInterceptor: 未知 domain 直接回 403，只有 localhost/127.0.0.1 可 fallback
- 加入租戶 status 檢查（info == null 時拒絕）
- 驗證: `Host: fake.example.com` → `{"code":403,"message":"Unknown tenant"}` ✅

### ✅ Fix 3: 每個租戶獨立 DB 帳號
- 建立 `ent_user` 帳號，只有 ent_mall 的 SELECT/INSERT/UPDATE/DELETE 權限
- 更新 ent_master.tenant 表的 db_username/db_password
- 密碼已存於 `/tmp/ent_user_pass.txt`
- 更新 `create-tenant.sh`: 自動建立專用帳號、隨機密碼、隨機管理員密碼

### ✅ Fix 4: MySQL root 限制 IP
- 刪除 `root@%` 和 `reader@%`
- 建立 `root@172.18.%`（只允許 Docker 內網）
- 保留 `root@localhost`（容器內管理用）
- Docker 容器間連線正常 ✅

### ✅ Fix 5: Actuator 限制存取
- 在 `ent-mall` 和 `mall-admin` nginx 配置加入 `location ~* /actuator { return 403; }`
- 驗證: actuator 端點回 403 ✅

### ✅ Fix 6: Nginx wildcard 排除 dev
- 確認 `*.homely-go.com` wildcard 配置（ent-saas）未啟用
- 實際使用的是 `ent-mall`（精確匹配 `ent.homely-go.com`）
- `mall-admin` 使用 `server_name _` 作為 default server
- dev.homely-go.com 會正確落入 default server ✅

### ✅ Fix 7: tenant/info 不回傳 tenantCode
- 從 Portal 和 Admin 的 TenantController 移除 `tenantCode` 欄位
- 前端無使用 tenantCode（grep 確認）
- 驗證: `/tenant/info` 只回傳 brandName, themeColor, brandLogo ✅

## 驗收結果
1. ✅ mvn build SUCCESS（mall-common, mall-mbg, mall-security, mall-admin, mall-portal）
2. ✅ Docker 容器重啟成功
3. ✅ nginx reload 成功
4. ✅ 所有 curl 測試通過

## 修改檔案清單
- `mall-portal/.../TenantController.java` — 移除 refresh 端點和 tenantCode
- `mall-admin/.../TenantController.java` — 移除 tenantCode
- `mall-admin/src/main/resources/application.yml` — 白名單 `/tenant/**` → `/tenant/info`
- `mall-common/.../TenantInterceptor.java` — 關閉 fallback，加 403 拒絕
- `scripts/create-tenant.sh` — 專用 DB 帳號、隨機密碼
- `/etc/nginx/sites-available/ent-mall` — 加 actuator 阻擋
- `/etc/nginx/sites-available/mall-admin` — 加 actuator 阻擋
- MySQL: 新增 ent_user, 刪除 root@%, reader@%, 新增 root@172.18.%
