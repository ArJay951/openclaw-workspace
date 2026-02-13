# TASK-003: SaaS 多租戶邏輯修復 - 完成報告

## 完成時間: 2026-02-12 13:20 UTC

## 修復項目狀態

### ✅ Fix 1: DataSource 動態新增
- **檔案**: `mall-common/.../TenantDataSourceConfig.java`
- **修改**: `refreshTenants()` 現在會比對現有 DataSource 與 registry，動態新增新租戶、移除停用租戶的連線池

### ✅ Fix 2: JWT Token 綁定租戶
- **檔案**: `mall-security/.../JwtTokenUtil.java`, `mall-security/.../JwtAuthenticationTokenFilter.java`
- **修改**:
  - `generateToken()` 加入 `tenant` claim
  - 新增 `getTenantFromToken()` 方法
  - JwtFilter 中從 Host header 自行解析 tenant（不依賴 TenantInterceptor）
  - 舊 token（不含 tenant claim）相容，不會被拒絕
  - Token tenant 與請求 tenant 不匹配時，不設定 authentication（回 401）
- **驗證**: JWT payload 確認包含 `"tenant":"ent"`

### ✅ Fix 3: RabbitMQ 租戶隔離
- **檔案**: `mall-portal/.../CancelOrderSender.java`, `mall-portal/.../CancelOrderReceiver.java`
- **修改**: 
  - 發送端: 在 MessagePostProcessor 中加入 `tenantCode` header
  - 消費端: 從 message header 取出 tenantCode，設定 TenantContext，finally 清除

### ✅ Fix 4: MongoDB 租戶隔離
- **檔案**: 3 個 domain class + 3 個 repository + 3 個 service impl
- **修改**:
  - `MemberReadHistory`, `MemberProductCollection`, `MemberBrandAttention` 加入 `@Indexed tenantCode` 欄位
  - Repository 新增帶 tenantCode 的查詢方法
  - Service 在新增時設定 tenantCode，查詢時帶 tenantCode 條件（有 fallback）

### ✅ Fix 5: 定時任務遍歷所有租戶
- **檔案**: `mall-portal/.../OrderTimeOutCancelTask.java`
- **修改**: 注入 TenantRegistry，遍歷所有租戶執行取消操作（@Component 仍被註解掉）

### ✅ Fix 6: Elasticsearch 租戶隔離（預備）
- **檔案**: `mall-search/.../EsProduct.java`
- **修改**: 加入 `@Field(type = FieldType.Keyword) tenantCode` 欄位
- **備註**: mall-search 未啟用，僅預備修改，未驗證

### ✅ Fix 7: Redis Hash Key 租戶前綴
- **檔案**: `mall-common/.../BaseRedisConfig.java`
- **修改**: `setHashKeySerializer` 從 `StringRedisSerializer` 改為 `TenantKeySerializer`

## 驗收結果

| 項目 | 結果 |
|------|------|
| mvn clean install (mall-common,mall-security,mall-admin,mall-portal) | ✅ BUILD SUCCESS |
| Docker 重啟 | ✅ 正常啟動 |
| Portal 首頁 (ent.homely-go.com) | ✅ code: 200 |
| Admin health | ✅ UP |
| JWT 包含 tenant claim | ✅ `"tenant":"ent"` |
| 舊 token 相容 | ✅ 不含 tenant 的 token 不會被拒絕 |

## 未驗證項目
- RabbitMQ 消息 tenantCode header（需觸發訂單取消流程）
- MongoDB tenantCode 欄位（需新增收藏/瀏覽記錄）
- Redis hash key 前綴（需操作 hash 類型的 cache）
- 跨租戶 token 拒絕（需要第二個租戶 domain）
- mall-search build（跳過）
