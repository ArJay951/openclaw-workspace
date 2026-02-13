# TASK-003: SaaS 多租戶邏輯問題修復

## 目標
修復多租戶架構中的 7 個邏輯問題，確保租戶間資料完全隔離。

## 修復項目

### Fix 1: DataSource 動態新增（不需重啟）

**問題**: 新租戶開通後，TenantRegistry.refresh() 只刷新 domain 映射，不會建立新的 DataSource 連線池。

**修復**: 在 TenantDataSourceConfig 的 refreshTenants() 中，檢查是否有新租戶需要建立 DataSource，有的話動態新增到 TenantDataSource。

**檔案**: `/home/ubuntu/entertainment-mall/mall-common/src/main/java/com/macro/mall/common/tenant/TenantDataSourceConfig.java`

修改 refreshTenants() 方法：
```java
@Scheduled(fixedDelay = 300000)
public void refreshTenants() {
    try {
        TenantRegistry registry = tenantRegistry();
        registry.refresh();
        
        // 動態新增/移除 DataSource
        Map<Object, Object> currentTargets = new HashMap<>(dynamicDataSource.getResolvedDataSources());
        Map<String, TenantRegistry.TenantInfo> allTenants = registry.getAllTenants();
        
        boolean changed = false;
        
        // 新增新租戶的 DataSource
        for (Map.Entry<String, TenantRegistry.TenantInfo> entry : allTenants.entrySet()) {
            if (!currentTargets.containsKey(entry.getKey())) {
                DataSource ds = createTenantDataSource(entry.getValue());
                currentTargets.put(entry.getKey(), ds);
                changed = true;
                log.info("Added datasource for new tenant: {}", entry.getKey());
            }
        }
        
        // 移除已停用租戶的 DataSource（可選，關閉連線池）
        Set<Object> toRemove = new HashSet<>();
        for (Object key : currentTargets.keySet()) {
            if (!allTenants.containsKey(key.toString())) {
                toRemove.add(key);
                changed = true;
                log.info("Removing datasource for disabled tenant: {}", key);
            }
        }
        for (Object key : toRemove) {
            Object removed = currentTargets.remove(key);
            if (removed instanceof DruidDataSource) {
                ((DruidDataSource) removed).close();
            }
        }
        
        if (changed) {
            dynamicDataSource.setTargetDataSources(currentTargets);
            dynamicDataSource.afterPropertiesSet();
            log.info("Dynamic datasource refreshed, now {} tenants", currentTargets.size());
        }
    } catch (Exception e) {
        log.warn("Failed to refresh tenants", e);
    }
}
```

注意：`AbstractRoutingDataSource.getResolvedDataSources()` 回傳的是 unmodifiable map，需要複製一份再操作。

**驗證**: 
1. 透過 create-tenant.sh 建立新租戶
2. 等 5 分鐘（或手動呼叫 admin refresh）
3. 新租戶 domain 可以存取 API，不需要重啟

### Fix 2: JWT Token 綁定租戶

**問題**: JWT token 只包含 username，不包含 tenantCode。租戶 A 的 token 可用在租戶 B。

**修復方案**: 在 JWT 生成時加入 tenantCode claim，驗證時檢查 token 中的 tenantCode 是否與當前請求的 tenantCode 一致。

**檔案**:
1. `/home/ubuntu/entertainment-mall/mall-security/src/main/java/com/macro/mall/security/util/JwtTokenUtil.java`
2. `/home/ubuntu/entertainment-mall/mall-security/src/main/java/com/macro/mall/security/component/JwtAuthenticationTokenFilter.java`

**JwtTokenUtil.java 修改**:

找到 generateToken 方法，加入 tenantCode：
```java
// 在 generateToken 中加入 tenant claim
public String generateToken(UserDetails userDetails) {
    Map<String, Object> claims = new HashMap<>();
    claims.put(CLAIM_KEY_USERNAME, userDetails.getUsername());
    claims.put(CLAIM_KEY_CREATED, new Date());
    // 加入租戶資訊
    String tenantCode = TenantContext.getTenantCode();
    if (tenantCode != null) {
        claims.put("tenant", tenantCode);
    }
    return generateToken(claims);
}
```

新增方法：
```java
public String getTenantFromToken(String token) {
    try {
        Claims claims = getClaimsFromToken(token);
        return claims.get("tenant", String.class);
    } catch (Exception e) {
        return null;
    }
}
```

**JwtAuthenticationTokenFilter.java 修改**:

在 doFilterInternal 中，驗證 token 的 tenant 是否與當前請求的 tenant 一致：
```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ... {
    String authHeader = request.getHeader(this.tokenHeader);
    if (authHeader != null && authHeader.startsWith(this.tokenHead)) {
        String authToken = authHeader.substring(this.tokenHead.length());
        String username = jwtTokenUtil.getUserNameFromToken(authToken);
        
        // 驗證 tenant
        String tokenTenant = jwtTokenUtil.getTenantFromToken(authToken);
        String currentTenant = TenantContext.getTenantCode();
        if (tokenTenant != null && currentTenant != null && !tokenTenant.equals(currentTenant)) {
            // tenant 不匹配，拒絕
            LOGGER.warn("Token tenant '{}' does not match current tenant '{}'", tokenTenant, currentTenant);
            chain.doFilter(request, response);
            return; // 不設定 authentication，後面會被 Spring Security 攔截回 401
        }
        
        // ... 原有的 username 驗證邏輯 ...
    }
    chain.doFilter(request, response);
}
```

需要 import `com.macro.mall.common.tenant.TenantContext`。

注意：TenantInterceptor 已經在 Spring MVC 層設定了 TenantContext，但 JwtAuthenticationTokenFilter 是 Spring Security Filter，執行順序在 Interceptor 之前。所以需要在 Filter 中也解析 tenant。

解法：在 JwtAuthenticationTokenFilter 的 doFilterInternal 開頭也做一次 tenant 解析（從 Host header），或者從 TenantContext 取值（如果 TenantInterceptor 已先執行的話）。

**更穩妥的做法**：在 JwtAuthenticationTokenFilter 中直接從 request.getServerName() 解析 tenant，不依賴 TenantContext：
```java
String host = request.getServerName();
// 需要注入 TenantRegistry
String currentTenant = tenantRegistry.getTenantCodeByDomain(host);
if (currentTenant == null && ("localhost".equals(host) || "127.0.0.1".equals(host))) {
    currentTenant = "ent"; // default
}
```

**驗證**: 
1. 用租戶 A 登入取得 token
2. 帶 A 的 token 存取 B 的 domain → 應回 401

### Fix 3: RabbitMQ 租戶隔離

**問題**: 所有租戶共用同一個 queue/exchange，訂單取消消息會混。

**修復方案**: Queue 名稱加上租戶前綴。

**檔案**: `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java`

查看現有 QueueEnum 定義，目前 queue 名稱是固定的。

**修改方式**：不改 QueueEnum（靜態值難以動態化），而是在發送消息時加入 tenantCode 到 message header，消費時檢查 tenantCode 並設定 TenantContext。

**發送端**（找到 sendMessage 的地方）：
```java
// 發送訂單取消消息時帶上 tenantCode
MessageProperties props = new MessageProperties();
props.setHeader("tenantCode", TenantContext.getTenantCode());
Message message = new Message(orderId.toString().getBytes(), props);
amqpTemplate.send(exchange, routeKey, message);
```

或者如果用的是 `rabbitTemplate.convertAndSend`，用 MessagePostProcessor：
```java
rabbitTemplate.convertAndSend(exchange, routeKey, orderId, message -> {
    message.getMessageProperties().setHeader("tenantCode", TenantContext.getTenantCode());
    return message;
});
```

**消費端**（找到 @RabbitListener 或 handleMessage 的地方）：
```java
@RabbitListener(queues = "...")
public void handle(Message message, Channel channel) {
    String tenantCode = (String) message.getMessageProperties().getHeaders().get("tenantCode");
    if (tenantCode != null) {
        TenantContext.setTenantCode(tenantCode);
    }
    try {
        // 原有邏輯
    } finally {
        TenantContext.clear();
    }
}
```

先找到 portal 中 RabbitMQ 發送和消費的具體程式碼再修改。

**驗證**: 觀察日誌中訂單取消操作是否帶有 tenantCode。

### Fix 4: MongoDB 租戶隔離

**問題**: `@Document` 沒有區分租戶，所有租戶的閱讀歷史、收藏等混在同一個 collection。

**修復方案**: 在 MongoDB document 中加入 tenantCode 欄位，查詢時加上 tenantCode 條件。

**檔案**:
- `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/domain/MemberReadHistory.java`
- `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/domain/MemberProductCollection.java`
- `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/domain/MemberBrandAttention.java`
- 對應的 Repository 和 Service 實作

**每個 domain class 加入**:
```java
@Indexed
private String tenantCode;
```

**每個 Service 的新增方法中加入**:
```java
entity.setTenantCode(TenantContext.getTenantCode());
```

**每個 Repository 的查詢方法改為帶 tenantCode**:
```java
List<MemberReadHistory> findByMemberIdAndTenantCodeOrderByCreateTimeDesc(Long memberId, String tenantCode);
```

或使用 `@Query` 加上 tenantCode 條件。

**驗證**: 新增一筆收藏，檢查 MongoDB document 是否包含 tenantCode 欄位。

### Fix 5: 定時任務遍歷所有租戶

**問題**: OrderTimeOutCancelTask 沒有 TenantContext，如果啟用會不知道操作哪個租戶。

**修復**: 雖然目前被 @Component 註解掉了，但預防未來啟用，加上租戶遍歷邏輯。

**檔案**: `/home/ubuntu/entertainment-mall/mall-portal/src/main/java/com/macro/mall/portal/component/OrderTimeOutCancelTask.java`

```java
@Scheduled(cron = "0 0/10 * ? * ?")
private void cancelTimeOutOrder() {
    Map<String, TenantRegistry.TenantInfo> tenants = tenantRegistry.getAllTenants();
    for (Map.Entry<String, TenantRegistry.TenantInfo> entry : tenants.entrySet()) {
        try {
            TenantContext.setTenantCode(entry.getKey());
            Integer count = portalOrderService.cancelTimeOutOrder();
            if (count > 0) {
                LOGGER.info("Tenant [{}]: 取消超時訂單 {} 筆", entry.getKey(), count);
            }
        } catch (Exception e) {
            LOGGER.error("Tenant [{}]: 取消訂單失敗", entry.getKey(), e);
        } finally {
            TenantContext.clear();
        }
    }
}
```

需要注入 TenantRegistry。注意此 class 目前 @Component 被註解掉了，保持被註解掉的狀態即可，只是改造內部邏輯預備未來。

### Fix 6: Elasticsearch 租戶隔離（預備）

**問題**: mall-search 的 ES index 沒有租戶區分。

**現狀**: mall-search 目前沒有啟用（沒有 Docker container 在跑）。

**修復**: 在 EsProduct domain 加入 tenantCode 欄位，查詢時加上 tenantCode 條件。

先找到 EsProduct 的定義：
- `/home/ubuntu/entertainment-mall/mall-search/src/main/java/com/macro/mall/search/domain/EsProduct.java`

加入：
```java
private String tenantCode;
```

在 EsProductService 的 importAll、search 等方法中加入 tenantCode 條件。

由於 mall-search 未啟用，只做預備修改即可，不需要驗證。如果 mall-search build 有問題可以跳過。

### Fix 7: Redis Hash Key 租戶前綴

**問題**: BaseRedisConfig 中 `setHashKeySerializer` 用的是原生 `StringRedisSerializer`，不是 `TenantKeySerializer`。

**檔案**: `/home/ubuntu/entertainment-mall/mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java`

**修改**:
```java
// 原來
redisTemplate.setHashKeySerializer(new StringRedisSerializer());
// 改為
redisTemplate.setHashKeySerializer(new TenantKeySerializer());
```

**驗證**: 檢查 Redis 中的 hash key 是否帶有租戶前綴。

## 驗收標準

1. ✅ 後端 `cd /home/ubuntu/entertainment-mall && mvn clean install -pl mall-common,mall-security,mall-admin,mall-portal -am -DskipTests` build 成功
2. ✅ Docker 重啟後服務正常啟動
3. ✅ 現有 ent.homely-go.com 功能正常（首頁、搜尋、登入）
4. ✅ JWT token 包含 tenant claim
5. ✅ RabbitMQ 消息包含 tenantCode header
6. ✅ MongoDB document 包含 tenantCode 欄位

## ⚠️ 注意事項

- **JwtAuthenticationTokenFilter 的執行順序**: Security Filter 在 MVC Interceptor 之前，注意 TenantContext 可能尚未設定
- **不要破壞現有登入流程**: 舊 token（不含 tenant claim）需要相容，不能直接拒絕
- **mall-search build 如果失敗可以跳過**（未啟用）
- **修改 JwtTokenUtil 需要同時改 mall-security 模組**
- **RabbitMQ 修改要找到正確的發送/消費程式碼位置**
- 後端專案路徑: `/home/ubuntu/entertainment-mall/`
- Docker Compose: `cd /home/ubuntu/entertainment-mall && docker compose up -d`
- 重建 Docker image: 先 mvn build，再 `docker build` admin 和 portal
