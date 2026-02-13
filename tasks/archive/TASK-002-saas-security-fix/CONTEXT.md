# CONTEXT - SaaS 資安修復

## 專案路徑
- 後端: `/home/ubuntu/entertainment-mall/`
- 前台: `/home/ubuntu/ent-mall-app-web/`
- 後台: `/home/ubuntu/ent-mall-admin-web/`

## 關鍵檔案
- Tenant Interceptor: `mall-common/src/main/java/com/macro/mall/common/tenant/TenantInterceptor.java`
- Tenant DataSource Config: `mall-common/src/main/java/com/macro/mall/common/tenant/TenantDataSourceConfig.java`
- Tenant Registry: `mall-common/src/main/java/com/macro/mall/common/tenant/TenantRegistry.java`
- Portal Tenant Controller: `mall-portal/src/main/java/com/macro/mall/portal/controller/TenantController.java`
- Admin Tenant Controller: `mall-admin/src/main/java/com/macro/mall/controller/TenantController.java`
- Create Tenant Script: `scripts/create-tenant.sh`
- Nginx: `/etc/nginx/sites-available/ent-saas`

## Docker
- MySQL container: `mall-mysql`
- Docker network: `general-mall_default` (172.18.0.0/16)
- 重啟指令: `cd /home/ubuntu/entertainment-mall && docker compose down && docker compose up -d`
- Build: `cd /home/ubuntu/entertainment-mall && mvn clean package -DskipTests` (排除 mall-demo, mall-search)

## DB
- Master DB: `ent_master`
- Tenant DB: `ent_mall`
- MySQL root password: `root`

## Nginx
- 通用商城: `/etc/nginx/sites-available/mall-admin` (server_name dev.homely-go.com)
- SaaS: `/etc/nginx/sites-available/ent-saas` (server_name *.homely-go.com)
- 重載: `sudo nginx -t && sudo systemctl reload nginx`

## 前端 tenant 引用
- `src/App.vue` 中 `this.$store.state.tenantInfo.tenantCode` — 檢查是否實際使用
- `src/store/index.js` — tenantInfo state
- `src/api/tenant.js` — fetchTenantInfo
