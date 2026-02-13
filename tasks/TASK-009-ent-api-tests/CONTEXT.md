# CONTEXT.md — TASK-009

## Entertainment Mall 架構

### Ports
- Admin: 8090
- Portal: 8095

### 域名
- ent.homely-go.com（第一個租戶）
- 多租戶需要 Host header 帶域名

### 後端路徑
- `/home/ubuntu/entertainment-mall/`

### Docker
- MySQL 容器: `ent-mall-mysql`（或查 docker ps）
- 連線: 需確認容器名

### Admin 登入
- `POST /admin/login` — body: `{ "username": "admin", "password": "123456" }`
- 回傳: `{ "data": { "tokenHead": "Bearer ", "token": "xxx" } }`

### 多租戶 Header
- Portal API 需帶 `Host: ent.homely-go.com` 才能正確解析租戶
- 本機測試用 curl -H "Host: ent.homely-go.com" http://127.0.0.1:8095/...

### 資安修復（已完成）
- /tenant/refresh Portal 已移除（404）
- /tenant/refresh Admin 需認證（401）
- 未知 domain → 403
- localhost/127.0.0.1 可 fallback
- actuator 被 Nginx 層擋 403
- tenant/info 不回傳 tenantCode

### Nginx
- Actuator 阻擋在 `/etc/nginx/sites-available/ent-mall`
- 測試 actuator 時要透過 Nginx（用域名或 Nginx port），不是直連 Java port

### DB
- `ent_mall`（租戶資料）
- `ent_master`（租戶註冊表）

### 注意
- Portal API 直連 8095 時 Host header 很重要，沒帶或帶錯會 403
- Admin API 直連 8090 不需要特殊 Host header（admin 有自己的租戶解析）
- 測試以只讀為主，不要建立/修改資料
