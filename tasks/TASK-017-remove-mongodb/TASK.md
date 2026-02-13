# TASK-017: 移除 MongoDB 依賴

## 目標
General Mall 不使用 MongoDB，移除相關程式碼和依賴，加快啟動速度。

## 要移除的檔案

### Controller（3個）
- `mall-portal/src/main/java/com/macro/mall/portal/controller/MemberAttentionController.java`
- `mall-portal/src/main/java/com/macro/mall/portal/controller/MemberProductCollectionController.java`
- `mall-portal/src/main/java/com/macro/mall/portal/controller/MemberReadHistoryController.java`

### Service Interface（3個）
- `mall-portal/src/main/java/com/macro/mall/portal/service/MemberAttentionService.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/MemberCollectionService.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/MemberReadHistoryService.java`

### Service Impl（3個）
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberAttentionServiceImpl.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberCollectionServiceImpl.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberReadHistoryServiceImpl.java`

### Repository（3個）
- `mall-portal/src/main/java/com/macro/mall/portal/repository/MemberReadHistoryRepository.java`
- `mall-portal/src/main/java/com/macro/mall/portal/repository/MemberProductCollectionRepository.java`
- `mall-portal/src/main/java/com/macro/mall/portal/repository/MemberBrandAttentionRepository.java`

### Domain（3個）
- `mall-portal/src/main/java/com/macro/mall/portal/domain/MemberReadHistory.java`
- `mall-portal/src/main/java/com/macro/mall/portal/domain/MemberProductCollection.java`
- `mall-portal/src/main/java/com/macro/mall/portal/domain/MemberBrandAttention.java`

## 配置修改

### 1. 排除 MongoDB Auto-Configuration
檔案: `mall-portal/src/main/java/com/macro/mall/portal/MallPortalApplication.java`

在 `@SpringBootApplication` 加 exclude：
```java
@SpringBootApplication(exclude = {
    org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration.class,
    org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration.class,
    org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration.class
})
```

### 2. 移除 application.yml 中 MongoDB 配置
- `application.yml` 裡的 `mongo:` 段落
- `application-dev.yml` 裡的 `spring.data.mongodb` 段落
- `application-prod.yml` 裡的 `spring.data.mongodb` 段落

### 3. 移除 pom.xml 中 MongoDB 依賴
檔案: `mall-portal/pom.xml`
移除 `spring-boot-starter-data-mongodb` 依賴

### 4. Docker Compose
檔案: `/home/ubuntu/general-mall/docker-compose.yml`
- 移除 `mongo` service
- 移除 `mongo_data` volume
- 移除 mall-portal environment 裡的 `MONGO_HOST`
- 移除 mall-portal 的 `depends_on` 裡的 mongo（如果有）

## Build & Deploy
```bash
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-portal/target && docker build -t mall/mall-portal:1.0-SNAPSHOT -f /tmp/Dockerfile-portal .
cd /home/ubuntu/general-mall && docker compose up -d mall-portal
# 停掉 mongo 容器
docker stop mall-mongo && docker rm mall-mongo
```

## 驗收標準
1. `mvn clean package -DskipTests` 成功
2. Portal 啟動成功，日誌無 MongoDB 相關訊息
3. 代收代付 API 正常（用 curl 測試）
4. `docker ps` 沒有 mall-mongo 容器
5. 啟動時間應明顯縮短（記錄 before/after）

## ⚠️ 注意
- Dockerfile: `/tmp/Dockerfile-portal`
- 只動 mall-portal，mall-admin 不動
- 刪除 Java 檔用 `rm`（不是 trash，因為是 git tracked）
- Git commit 用中文
- 先記錄當前啟動時間（日誌裡 `Started MallPortalApplication in X seconds`）
