# TASK-002: SaaS 多租戶資安修復

## 目標
修復 TASK-001 多租戶改造中的資安風險。

## 修復項目

### 🔴 Fix 1: /tenant/refresh 加認證
**問題**: 任何人都可以呼叫 POST /tenant/refresh，觸發數據源重載。
**修復**:
- Portal 的 TenantController: 移除 /tenant/refresh 端點（portal 不需要手動 refresh，已有 5 分鐘定時刷新）
- Admin 的 TenantController: /tenant/refresh 需要 admin 登入 token 才能呼叫（已有 Spring Security，確認此路徑不在白名單中）

**檔案**:
- `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/controller/TenantController.java` — 刪除 refresh 端點
- `/home/ubuntu/entertainment-mall/mall-admin/src/main/java/com/macro/mall/controller/TenantController.java` — 保留 refresh，確認需要認證

**驗證**: `curl -s -X POST http://localhost:8095/tenant/refresh` 應該回 404 或 405。Admin 端不帶 token 呼叫應回 401。

### 🔴 Fix 2: 關閉 domain fallback
**問題**: TenantInterceptor 中 domain 找不到時 fallback 到 default tenant，攻擊者可用任意 domain 存取資料。
**修復**: 找不到租戶時直接回 403，不 fallback。但保留 localhost / 127.0.0.1 的 fallback（開發用）。

**檔案**:
- `/home/ubuntu/entertainment-mall/mall-common/src/main/java/com/macro/mall/common/tenant/TenantInterceptor.java`

修改邏輯:
```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    String host = request.getServerName();
    String tenantCode = tenantRegistry.getTenantCodeByDomain(host);
    
    // 只允許 localhost 和 127.0.0.1 使用 fallback
    if (tenantCode == null) {
        if ("localhost".equals(host) || "127.0.0.1".equals(host)) {
            tenantCode = defaultTenantCode;
        } else {
            response.setStatus(403);
            response.setContentType("application/json;charset=UTF-8");
            try {
                response.getWriter().write("{\"code\":403,\"message\":\"Unknown tenant\"}");
            } catch (Exception e) {
                log.error("Failed to write 403 response", e);
            }
            return false;
        }
    }
    
    // 檢查租戶是否已過期或停用
    TenantRegistry.TenantInfo info = tenantRegistry.getTenantInfo(tenantCode);
    if (info == null || info.status != 1) {
        response.setStatus(403);
        try {
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write("{\"code\":403,\"message\":\"Tenant disabled\"}");
        } catch (Exception e) {
            log.error("Failed to write response", e);
        }
        return false;
    }
    
    TenantContext.setTenantCode(tenantCode);
    return true;
}
```

注意: TenantRegistry.TenantInfo 需要加 `public int status;` 欄位（如果還沒有的話），並在 refresh() 中從 ResultSet 讀取 status。

**驗證**: `curl -s http://localhost:8095/home/content -H "Host: fake.homely-go.com"` 應回 403。

### 🔴 Fix 3: 每個租戶獨立 DB 帳號
**問題**: 所有租戶用 root，無最小權限。
**修復**: 
1. 建立 ent_mall 的專用帳號
2. 更新 tenant 表的 db_username/db_password
3. 修改 create-tenant.sh，自動建立專用帳號

**執行 SQL**:
```sql
-- 為現有 ent_mall 租戶建立專用帳號
CREATE USER 'ent_user'@'%' IDENTIFIED BY '自動生成的隨機密碼';
GRANT SELECT, INSERT, UPDATE, DELETE ON ent_mall.* TO 'ent_user'@'%';
FLUSH PRIVILEGES;

-- 更新 tenant 表
UPDATE ent_master.tenant SET db_username='ent_user', db_password='同上密碼' WHERE tenant_code='ent';
```

**修改 create-tenant.sh**: 每次開通新租戶時自動建立 `{tenant_code}_user` 帳號，只給該 DB 的 CRUD 權限。密碼用 `openssl rand -base64 16` 隨機生成。

同時在 create-tenant.sh 中：
- 管理員預設密碼改為隨機生成，腳本結束時印出
- 或至少強制首次登入改密碼

### 🔴 Fix 4: MySQL root 限制 IP
**問題**: `root@%` 允許任意 IP 連線。
**修復**:
```sql
-- 刪除 root@% (只保留 root@localhost)
DROP USER 'root'@'%';
-- 同時刪除 reader@%
DROP USER 'reader'@'%';
FLUSH PRIVILEGES;
```

注意: Docker 容器內連線用的是 Docker 內網 IP，確認 mall-mysql container 內 root@localhost 仍可用（容器內的應用透過 socket 或 localhost 連線）。

**重要**: 先確認所有 Spring Boot 應用是透過 Docker 網路（mall-mysql hostname）連線，而非透過 root@%。如果 Docker 容器之間連線需要 root@'172.18.%'，就改為限制 Docker 網段而非 %。

做法:
```sql
-- 先檢查 Docker 網段
-- Docker network 是 172.18.0.0/16
-- 改為只允許 Docker 內網
CREATE USER 'root'@'172.18.%' IDENTIFIED BY 'root';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'172.18.%' WITH GRANT OPTION;
DROP USER 'root'@'%';
FLUSH PRIVILEGES;
```

**驗證**: 從伺服器外部連 3306 應被拒。Docker 容器內連線正常。

### 🟡 Fix 5: Actuator 限制存取
**問題**: /actuator/health 可被外部存取。
**修復**: 在 Nginx 層擋掉。

在 `/etc/nginx/sites-available/ent-saas` 和其他相關 nginx 配置加:
```nginx
location /actuator/ {
    deny all;
    return 403;
}

location /admin-api/actuator/ {
    deny all;
    return 403;
}

location /portal-api/actuator/ {
    deny all;
    return 403;
}
```

### 🟡 Fix 6: Nginx wildcard 排除 dev
**問題**: `*.homely-go.com` 會攔截 dev.homely-go.com（通用商城）。
**修復**: 確認 `/etc/nginx/sites-available/mall-admin`（通用商城）的 server_name 有 `dev.homely-go.com` 且優先於 wildcard。

Nginx 匹配順序: 精確 > wildcard 開頭 > wildcard 結尾 > regex。所以 `dev.homely-go.com` 精確匹配會優先於 `*.homely-go.com`。

**驗證**: `curl -s http://dev.homely-go.com` 確認還是通用商城。

### 🟡 Fix 7: tenant/info 不回傳 tenantCode
**問題**: 回傳 tenantCode 可被列舉。
**修復**: 從 /tenant/info 回傳中移除 tenantCode 欄位（只回傳 brandName, brandLogo, themeColor）。

**檔案**: 兩個 TenantController.java，移除 `result.put("tenantCode", ...)` 行。

但前端可能有用到 tenantCode，先檢查前端是否有引用 tenantCode，如果有就保留。

## 驗收標準
1. ✅ `curl -X POST http://localhost:8095/tenant/refresh` → 404/405
2. ✅ `curl http://localhost:8095/home/content -H "Host: fake.example.com"` → 403
3. ✅ `curl http://localhost:8095/home/content -H "Host: ent.homely-go.com"` → 200（正常）
4. ✅ ent_mall 使用專用帳號而非 root
5. ✅ Actuator 從外部不可存取
6. ✅ dev.homely-go.com 仍正常
7. ✅ 後端 mvn build 成功
8. ✅ Docker 重啟後服務正常

## ⚠️ 注意
- 改 MySQL 使用者前先備份
- 改完 DB 帳號後要重啟後端容器
- 不要鎖死自己（確保至少一個 root 帳號可用）
- 測試 Docker 容器間連線是否正常
