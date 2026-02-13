# TASK-001: 娛樂城商城 SaaS 多租戶改造

## 目標
把現有娛樂城商城系統改造為多租戶 SaaS 架構，一套程式碼服務多個客戶。

## 決策
- 現有 ent.homely-go.com 作為第一個租戶
- 前台域名支援子域名 + 自訂域名
- 後台域名格式：`{tenant}-admin.homely-go.com`
- 資料隔離策略：每個租戶獨立 database

## 步驟

### Step 1: Master DB 建立

在現有 MySQL（mall-mysql container）中建立 `ent_master` 資料庫：

```sql
CREATE DATABASE ent_master DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 租戶表
CREATE TABLE ent_master.tenant (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_code VARCHAR(50) NOT NULL UNIQUE COMMENT '租戶代碼，如 ent, clientA',
    tenant_name VARCHAR(100) NOT NULL COMMENT '租戶名稱',
    -- 域名設定
    portal_domain VARCHAR(200) COMMENT '前台域名，如 ent.homely-go.com 或 shop.client.com',
    admin_domain VARCHAR(200) COMMENT '後台域名，如 ent-admin.homely-go.com',
    -- DB 設定
    db_name VARCHAR(100) NOT NULL COMMENT '資料庫名稱',
    db_host VARCHAR(200) DEFAULT 'localhost' COMMENT 'DB host',
    db_port INT DEFAULT 3306,
    db_username VARCHAR(100) DEFAULT 'root',
    db_password VARCHAR(200) DEFAULT 'root',
    -- 品牌設定
    brand_name VARCHAR(100) COMMENT '品牌名稱（顯示在前台）',
    brand_logo VARCHAR(500) COMMENT 'Logo URL',
    theme_color VARCHAR(20) DEFAULT '#b31e22' COMMENT '主題色',
    -- 狀態
    status TINYINT DEFAULT 1 COMMENT '0=停用 1=啟用',
    plan VARCHAR(20) DEFAULT 'basic' COMMENT '方案: basic/pro/premium',
    expired_at DATETIME COMMENT '到期時間',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_portal_domain (portal_domain),
    INDEX idx_admin_domain (admin_domain),
    INDEX idx_tenant_code (tenant_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 插入第一個租戶（現有系統）
INSERT INTO ent_master.tenant (tenant_code, tenant_name, portal_domain, admin_domain, db_name, brand_name, status)
VALUES ('ent', '娛樂城精品商城', 'ent.homely-go.com', 'ent-admin.homely-go.com', 'ent_mall', '娛樂城精品商城', 1);
```

### Step 2: 後端 - 動態數據源（核心改造）

修改 `/home/ubuntu/entertainment-mall/` Spring Boot 專案：

#### 2.1 新增依賴（mall-common/pom.xml 或 mall-portal/pom.xml）
不需額外依賴，Spring Boot 原生 `AbstractRoutingDataSource` 即可。

#### 2.2 新增類別

**TenantContext.java** — ThreadLocal 存當前租戶
```java
package com.macro.mall.common.tenant;

public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    
    public static void setTenantCode(String tenantCode) {
        CURRENT_TENANT.set(tenantCode);
    }
    
    public static String getTenantCode() {
        return CURRENT_TENANT.get();
    }
    
    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

**TenantDataSource.java** — 動態數據源路由
```java
package com.macro.mall.common.tenant;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class TenantDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getTenantCode();
    }
}
```

**TenantInterceptor.java** — 從 request 的 Host header 解析租戶
```java
package com.macro.mall.common.tenant;

import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class TenantInterceptor implements HandlerInterceptor {
    
    private final TenantRegistry tenantRegistry;
    
    public TenantInterceptor(TenantRegistry tenantRegistry) {
        this.tenantRegistry = tenantRegistry;
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String host = request.getServerName(); // e.g., ent.homely-go.com
        String tenantCode = tenantRegistry.getTenantCodeByDomain(host);
        if (tenantCode == null) {
            response.setStatus(404);
            return false;
        }
        TenantContext.setTenantCode(tenantCode);
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        TenantContext.clear();
    }
}
```

**TenantRegistry.java** — 從 master DB 載入租戶清單（啟動時載入 + 定時刷新）
```java
package com.macro.mall.common.tenant;

import javax.sql.DataSource;
import java.sql.*;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class TenantRegistry {
    private final DataSource masterDataSource;
    // domain -> tenantCode
    private final Map<String, String> domainMap = new ConcurrentHashMap<>();
    // tenantCode -> TenantInfo
    private final Map<String, TenantInfo> tenantMap = new ConcurrentHashMap<>();
    
    public TenantRegistry(DataSource masterDataSource) {
        this.masterDataSource = masterDataSource;
        refresh();
    }
    
    public void refresh() {
        try (Connection conn = masterDataSource.getConnection();
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM tenant WHERE status = 1")) {
            domainMap.clear();
            while (rs.next()) {
                TenantInfo info = new TenantInfo();
                info.tenantCode = rs.getString("tenant_code");
                info.dbName = rs.getString("db_name");
                info.dbHost = rs.getString("db_host");
                info.dbPort = rs.getInt("db_port");
                info.dbUsername = rs.getString("db_username");
                info.dbPassword = rs.getString("db_password");
                info.brandName = rs.getString("brand_name");
                info.brandLogo = rs.getString("brand_logo");
                info.themeColor = rs.getString("theme_color");
                
                tenantMap.put(info.tenantCode, info);
                
                String portalDomain = rs.getString("portal_domain");
                String adminDomain = rs.getString("admin_domain");
                if (portalDomain != null) domainMap.put(portalDomain, info.tenantCode);
                if (adminDomain != null) domainMap.put(adminDomain, info.tenantCode);
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to load tenants", e);
        }
    }
    
    public String getTenantCodeByDomain(String domain) {
        return domainMap.get(domain);
    }
    
    public TenantInfo getTenantInfo(String tenantCode) {
        return tenantMap.get(tenantCode);
    }
    
    public Map<String, TenantInfo> getAllTenants() {
        return tenantMap;
    }
    
    public static class TenantInfo {
        public String tenantCode;
        public String dbName;
        public String dbHost;
        public int dbPort;
        public String dbUsername;
        public String dbPassword;
        public String brandName;
        public String brandLogo;
        public String themeColor;
    }
}
```

**TenantDataSourceConfig.java** — 配置動態數據源
```java
package com.macro.mall.common.tenant;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class TenantDataSourceConfig {
    
    @Bean
    public DataSource masterDataSource() {
        // ⚠️ Docker 內用 mall-mysql，開發環境用 localhost
        // 建議從 application.yml 讀取 master DB 配置
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://mall-mysql:3306/ent_master?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Taipei&useSSL=false");
        ds.setUsername("root");
        ds.setPassword("root");
        ds.setMaximumPoolSize(5);
        return ds;
    }
    
    @Bean
    public TenantRegistry tenantRegistry() {
        return new TenantRegistry(masterDataSource());
    }
    
    @Bean
    @Primary
    public DataSource dataSource() {
        TenantDataSource dynamicDs = new TenantDataSource();
        Map<Object, Object> targetDataSources = new HashMap<>();
        
        TenantRegistry registry = tenantRegistry();
        for (Map.Entry<String, TenantRegistry.TenantInfo> entry : registry.getAllTenants().entrySet()) {
            TenantRegistry.TenantInfo info = entry.getValue();
            HikariDataSource ds = new HikariDataSource();
            ds.setJdbcUrl("jdbc:mysql://" + info.dbHost + ":" + info.dbPort + "/" + info.dbName 
                + "?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Taipei&useSSL=false");
            ds.setUsername(info.dbUsername);
            ds.setPassword(info.dbPassword);
            ds.setMaximumPoolSize(10);
            // ⚠️ 注意：現有專案用 Druid 連線池，這裡用 HikariCP 給 master 和動態源
            // 如果跟 Druid 衝突，需要排除 Druid 的自動配置或統一用 Druid
            targetDataSources.put(info.tenantCode, ds);
        }
        
        dynamicDs.setTargetDataSources(targetDataSources);
        // 預設用第一個租戶
        if (!targetDataSources.isEmpty()) {
            dynamicDs.setDefaultTargetDataSource(targetDataSources.values().iterator().next());
        }
        return dynamicDs;
    }
}
```

**TenantWebConfig.java** — 註冊攔截器
```java
package com.macro.mall.common.tenant;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class TenantWebConfig implements WebMvcConfigurer {
    
    @Autowired
    private TenantRegistry tenantRegistry;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TenantInterceptor(tenantRegistry))
                .addPathPatterns("/**")
                .excludePathPatterns("/actuator/**");
    }
}
```

#### 2.3 修改現有 DataSource 配置
- 移除或覆蓋原本 application.yml 中的 datasource 配置
- 讓 TenantDataSourceConfig 接管

#### 2.4 Redis Key 隔離
修改 Redis 配置，key 加上租戶前綴：
- 找到 RedisConfig 或 RedisTemplate 配置
- key serializer 加上 `TenantContext.getTenantCode() + ":"` 前綴

### Step 3: 前端適配

#### 3.1 Portal API 新增租戶資訊接口
在 portal 後端新增：
```
GET /tenant/info
回傳：{ brandName, brandLogo, themeColor }
```
不需要登入就能呼叫（前端載入時就要取得）。

#### 3.2 前台（ent-mall-app-web）
修改 `/home/ubuntu/ent-mall-app-web/src/App.vue`：
- 頁面載入時呼叫 `/portal-api/tenant/info`
- 把 brandName、logo、themeColor 存到 Vuex
- 用這些值動態替換 logo 文字、頁面標題

#### 3.3 後台（ent-mall-admin-web）
修改 `/home/ubuntu/ent-mall-admin-web/`：
- 同樣呼叫 tenant info API
- 動態替換 sidebar logo 和標題

### Step 4: 租戶開通腳本

建立 `/home/ubuntu/entertainment-mall/scripts/create-tenant.sh`：

```bash
#!/bin/bash
# 用法: ./create-tenant.sh <tenant_code> <tenant_name> <portal_domain> <admin_domain>

TENANT_CODE=$1
TENANT_NAME=$2
PORTAL_DOMAIN=$3
ADMIN_DOMAIN=$4
DB_NAME="ent_${TENANT_CODE}"

# 1. 複製資料庫
echo "Creating database ${DB_NAME}..."
docker exec mall-mysql mysqldump -uroot -proot ent_mall --no-data > /tmp/ent_template.sql
docker exec -i mall-mysql mysql -uroot -proot -e "CREATE DATABASE ${DB_NAME} DEFAULT CHARACTER SET utf8mb4;"
docker exec -i mall-mysql mysql -uroot -proot ${DB_NAME} < /tmp/ent_template.sql

# 2. 初始化管理員帳號（密碼: admin123）
docker exec -i mall-mysql mysql -uroot -proot ${DB_NAME} -e "
INSERT INTO ums_admin (username, password, email, nick_name, status, created_time)
VALUES ('admin', '\$2a\$10\$...hashed...', 'admin@${TENANT_CODE}.com', '管理員', 1, NOW());
"

# 3. 寫入 master DB
docker exec -i mall-mysql mysql -uroot -proot ent_master -e "
INSERT INTO tenant (tenant_code, tenant_name, portal_domain, admin_domain, db_name, brand_name, status)
VALUES ('${TENANT_CODE}', '${TENANT_NAME}', '${PORTAL_DOMAIN}', '${ADMIN_DOMAIN}', '${DB_NAME}', '${TENANT_NAME}', 1);
"

# 4. 呼叫後端 refresh 租戶（或重啟）
curl -X POST http://localhost:8095/tenant/refresh 2>/dev/null
curl -X POST http://localhost:8090/tenant/refresh 2>/dev/null

echo "Tenant ${TENANT_CODE} created!"
echo "Portal: ${PORTAL_DOMAIN}"
echo "Admin: ${ADMIN_DOMAIN}"
```

### Step 5: Nginx 配置

修改 nginx 支援 wildcard subdomain：

```nginx
# /etc/nginx/sites-available/ent-saas-portal
server {
    listen 80;
    server_name *.homely-go.com;  # wildcard
    
    # 前台靜態檔案
    location / {
        root /home/ubuntu/ent-mall-app-web/dist;
        try_files $uri $uri/ /index.html;
    }
    
    # Portal API 代理
    location /portal-api/ {
        proxy_pass http://localhost:8095/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# /etc/nginx/sites-available/ent-saas-admin
server {
    listen 80;
    server_name *-admin.homely-go.com;  # admin wildcard
    
    location / {
        root /home/ubuntu/ent-mall-admin-web/dist;
        try_files $uri $uri/ /index.html;
    }
    
    location /admin-api/ {
        proxy_pass http://localhost:8090/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 驗收標準

1. ✅ Master DB 建立成功，有第一個租戶資料
2. ✅ 後端可根據 domain 自動切換 DB
3. ✅ 現有 ent.homely-go.com 功能不受影響（作為第一個租戶）
4. ✅ 前台能動態顯示租戶品牌名稱
5. ✅ 開通腳本可以一鍵建立新租戶
6. ✅ `mvn clean package -DskipTests` 後端 build 成功
7. ✅ `npm run build` 前台前端 build 成功

## ⚠️ 注意事項

- **不要動現有的業務邏輯**，只加多租戶層
- **不要改 ent_mall 資料庫結構**
- 原本 application.yml 的 DB 配置要保留做 fallback
- Redis prefix 改動要確保不影響現有快取
- 阿杰偏好：不喜歡大面積強調色，UI 改動要低調
- charset 統一用 utf8mb4
