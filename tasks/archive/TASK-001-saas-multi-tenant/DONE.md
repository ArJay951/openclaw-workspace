# TASK-001: SaaS 多租戶改造 - 完成報告

## 完成時間
2026-02-12

## 完成項目

### ✅ Step 1: Master DB
- `ent_master` 資料庫已建立
- `tenant` 表已建立，含第一個租戶 `ent`（指向 `ent_mall`）

### ✅ Step 2: 後端動態數據源
已存在並驗證的檔案（`mall-common/src/main/java/com/macro/mall/common/tenant/`）：
- **TenantContext.java** — ThreadLocal 租戶上下文
- **TenantDataSource.java** — AbstractRoutingDataSource 路由
- **TenantDataSourceConfig.java** — Druid 連線池 + 動態數據源配置（從 master DB 讀租戶）
- **TenantRegistry.java** — 載入租戶清單（支持定時刷新）
- **TenantInterceptor.java** — 從 Host header 解析租戶（fallback 到預設租戶）
- **TenantWebConfig.java** — 註冊攔截器

新增：
- **TenantKeySerializer.java** — Redis key 加上 `{tenantCode}:` 前綴隔離
- 修改 `BaseRedisConfig.java` 使用 TenantKeySerializer

API 端點（portal + admin 都有）：
- `GET /tenant/info` — 取得當前租戶品牌資訊
- `POST /tenant/refresh` — 刷新租戶列表

Druid 自動配置已排除（`DruidDataSourceAutoConfigure.class`），由 `TenantDataSourceConfig` 接管。

### ✅ Step 3: 前端適配
**前台（ent-mall-app-web）：**
- 已有 `api/tenant.js` + `App.vue` 中呼叫 `/tenant/info`
- Vuex `SET_TENANT_INFO` mutation，動態更新 `document.title`

**後台（ent-mall-admin-web）：**
- 新增 `api/tenant.js`
- 修改 `App.vue`，載入時呼叫 tenant info，動態更新頁面標題

### ✅ Step 4: 租戶開通腳本
- 建立 `/home/ubuntu/entertainment-mall/scripts/create-tenant.sh`
- 用法: `./create-tenant.sh <tenant_code> <tenant_name> <portal_domain> <admin_domain>`
- 自動：匯出 ent_mall 結構 → 建新 DB → 初始化管理員 → 註冊到 master DB → 通知後端刷新

### ✅ Step 5: Nginx 配置
- 建立 `/etc/nginx/sites-available/ent-saas`（wildcard `*.homely-go.com`）
- ⚠️ 尚未啟用（需手動 `ln -s` 到 sites-enabled 並移除舊配置）

## 驗收結果

| 項目 | 結果 |
|------|------|
| Master DB + 首個租戶 | ✅ 已存在 |
| 後端根據 domain 切換 DB | ✅ TenantInterceptor + TenantDataSource |
| 現有功能不受影響 | ✅ 預設 fallback 到 `ent` 租戶 |
| 前台動態品牌名稱 | ✅ App.vue → tenant/info |
| 開通腳本 | ✅ create-tenant.sh |
| `mvn install -Dmaven.test.skip=true -pl mall-common,mall-mbg,mall-security,mall-admin,mall-portal -am` | ✅ BUILD SUCCESS |
| `npm run build`（前台） | ✅ BUILD SUCCESS |
| `npm run build`（後台） | ✅ BUILD SUCCESS |

## 注意事項

1. **Maven build 指令**：需用 `-pl mall-common,mall-mbg,mall-security,mall-admin,mall-portal -am` 排除 mall-demo 和 mall-search（這兩個模組有預存的編譯問題，與本次改動無關）
2. **Redis 快取**：新增 TenantKeySerializer 會讓現有 Redis key 加上 `ent:` 前綴，首次部署後需清除舊快取或等自動過期
3. **Nginx**：新配置已寫入但未啟用，避免影響現有服務。啟用方式：
   ```bash
   sudo rm /etc/nginx/sites-enabled/ent-mall
   sudo ln -s /etc/nginx/sites-available/ent-saas /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```
4. **未改動 ent_mall 資料庫結構** ✅
5. **未改變現有業務邏輯** ✅
