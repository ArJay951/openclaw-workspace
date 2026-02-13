# CONTEXT - SaaS 邏輯修復

## 專案路徑
- 後端: `/home/ubuntu/entertainment-mall/`
- mall-common: 共用模組（tenant 相關類別都在這）
- mall-security: 安全模組（JWT、Spring Security）
- mall-admin: 後台 API (port 8090 → container port 8080)
- mall-portal: 前台 API (port 8095 → container port 8085)
- mall-search: 搜尋服務（未啟用）

## 租戶相關類別位置
```
mall-common/src/main/java/com/macro/mall/common/tenant/
├── TenantContext.java         # ThreadLocal
├── TenantDataSource.java      # AbstractRoutingDataSource
├── TenantDataSourceConfig.java # DataSource 配置 + 定時刷新
├── TenantInterceptor.java     # HTTP 攔截器
├── TenantKeySerializer.java   # Redis key 序列化
├── TenantRegistry.java        # 租戶清單管理
└── TenantWebConfig.java       # 攔截器註冊
```

## JWT 相關
- JwtTokenUtil: `mall-security/src/main/java/com/macro/mall/security/util/JwtTokenUtil.java`
- JwtFilter: `mall-security/src/main/java/com/macro/mall/security/component/JwtAuthenticationTokenFilter.java`
- SecurityConfig: `mall-security/src/main/java/com/macro/mall/security/config/SecurityConfig.java`

## RabbitMQ 相關
- QueueEnum: `mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java`
- RabbitMqConfig: `mall-portal/src/main/java/com/macro/mall/portal/config/RabbitMqConfig.java`
- 找發送消息的程式碼: grep `amqpTemplate\|rabbitTemplate` in mall-portal/src/

## MongoDB 相關
- Domain classes: `mall-portal/src/main/java/com/macro/mall/portal/domain/Member*.java`
- Repository: `mall-portal/src/main/java/com/macro/mall/portal/repository/`
- Service: grep `MongoRepository\|memberReadHistory\|memberProductCollection\|memberBrandAttention` in mall-portal/src/

## Redis 相關
- BaseRedisConfig: `mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java`

## 定時任務
- OrderTimeOutCancelTask: `mall-portal/src/main/java/com/macro/mall/portal/component/OrderTimeOutCancelTask.java`
- 注意: 目前 @Component 被註解掉了

## Build 指令
```bash
cd /home/ubuntu/entertainment-mall
# Build (排除 mall-demo 和 mall-search 如果它們有問題)
mvn clean install -pl mall-common,mall-security,mall-admin,mall-portal -am -DskipTests

# 如果 mall-search 也要 build:
mvn clean install -pl mall-search -am -DskipTests || echo "mall-search build failed, skipping"
```

## Docker 部署
```bash
cd /home/ubuntu/entertainment-mall

# Rebuild images
cd mall-admin && docker build -t ent-mall/mall-admin:1.0-SNAPSHOT -f target/docker/ent-mall/mall-admin/1.0-SNAPSHOT/build/Dockerfile target/docker/ent-mall/mall-admin/1.0-SNAPSHOT/build/
cd ../mall-portal && docker build -t ent-mall/mall-portal:1.0-SNAPSHOT -f target/docker/ent-mall/mall-portal/1.0-SNAPSHOT/build/Dockerfile target/docker/ent-mall/mall-portal/1.0-SNAPSHOT/build/

# Restart
cd /home/ubuntu/entertainment-mall
docker compose down && docker compose up -d
```

## 測試
```bash
# 等待啟動 (約 30-40 秒)
sleep 40

# 測試首頁
curl -s http://localhost:8095/home/content -H "Host: ent.homely-go.com" | python3 -c "import sys,json; d=json.load(sys.stdin); print('code:', d.get('code'))"

# 測試登入
curl -s -X POST http://localhost:8095/sso/login -H "Host: ent.homely-go.com" -H "Content-Type: application/x-www-form-urlencoded" -d "username=test&password=123456"
```

## Master DB
- Database: ent_master
- Table: tenant
- 連線: docker exec mall-mysql mysql -uroot -proot ent_master

## 重要注意
- TenantInterceptor 在 Spring MVC 層執行
- JwtAuthenticationTokenFilter 在 Spring Security Filter 層執行（更早）
- 所以在 JwtFilter 中不能依賴 TenantContext（可能尚未設定）
- 需要在 JwtFilter 中自行從 Host header 解析 tenant
