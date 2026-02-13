# TASK-004: Auto-Order API 改造（Token 認證 + 廠商角色）

## 目標
改造 General Mall 的代收代付 API：
1. IP 白名單驗證 → Token 認證
2. 請求參數改為 employeeId + deviceId + amount
3. 後台管理員工/設備對應關係
4. 新增廠商角色，只能看自己來源的訂單

## ⚠️ 不做的事
- 不動商品選擇算法（selectProductsMultiRound）
- 不動訂單建立邏輯本身
- 不動前台 app（這是純 API 服務）

---

## Step 1: DB 改造

### 1a. 修改 `oms_order_source` 表
```sql
ALTER TABLE oms_order_source 
  ADD COLUMN api_token VARCHAR(64) NOT NULL COMMENT 'API Token' AFTER name,
  DROP COLUMN ip_whitelist;
```

### 1b. 新增 `oms_order_source_device` 表
```sql
CREATE TABLE oms_order_source_device (
  id BIGINT(20) NOT NULL AUTO_INCREMENT,
  source_id BIGINT(20) NOT NULL COMMENT '來源ID',
  employee_id VARCHAR(64) NOT NULL COMMENT '員工ID',
  device_id VARCHAR(64) NOT NULL COMMENT '設備號',
  status INT(1) DEFAULT 1 COMMENT '0=停用 1=啟用',
  create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_source_id (source_id),
  KEY idx_employee_device (employee_id, device_id),
  CONSTRAINT fk_device_source FOREIGN KEY (source_id) REFERENCES oms_order_source(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='來源設備管理';
```

### 1c. 修改 `oms_order` 表（加欄位）
```sql
ALTER TABLE oms_order
  ADD COLUMN employee_id VARCHAR(64) DEFAULT NULL COMMENT '員工ID' AFTER source_name,
  ADD COLUMN device_id VARCHAR(64) DEFAULT NULL COMMENT '設備號' AFTER employee_id;
```

### 1d. 修改 `ums_admin` 表（廠商綁定）
```sql
ALTER TABLE ums_admin
  ADD COLUMN source_id BIGINT(20) DEFAULT NULL COMMENT '綁定的來源ID（廠商用）';
```

### 1e. 為現有來源生成 Token
```sql
UPDATE oms_order_source SET api_token = MD5(CONCAT(id, '-', name, '-', UUID())) WHERE api_token IS NULL OR api_token = '';
```

---

## Step 2: Model 層 (mall-mbg)

### 2a. 更新 `OmsOrderSource.java`
- 移除 `ipWhitelist` 欄位
- 新增 `apiToken` (String) 欄位

### 2b. 新增 `OmsOrderSourceDevice.java`
- 欄位: id, sourceId, employeeId, deviceId, status, createTime, updateTime

### 2c. 新增 `OmsOrderSourceDeviceMapper.java`
- selectBySourceId(Long sourceId) → List
- selectByEmployeeAndDevice(String employeeId, String deviceId) → OmsOrderSourceDevice
- insert / update / delete
- selectAllEnabled() → List（帶 join source）

### 2d. 更新 `OmsOrderSourceMapper.java`
- 移除 ipWhitelist 相關映射
- 新增 apiToken 映射
- 新增 selectByApiToken(String token) → OmsOrderSource
- 新增 regenerateToken(Long id, String newToken)

### 2e. 更新 `OmsOrder.java`
- 新增 employeeId, deviceId 欄位

### 2f. 更新 `UmsAdmin.java`
- 新增 sourceId (Long) 欄位

---

## Step 3: Portal — API 認證改造

### 3a. 更新 `AutoOrderRequest.java`
```java
// 移除 name, phone
// 新增:
private String employeeId;  // @NotBlank
private String deviceId;    // @NotBlank
private BigDecimal amount;  // 保留
// 保留 sourceIp, sourceName (後端填入)
```

### 3b. 更新 `AutoOrderController.java`
- 認證邏輯改為：
  1. 從 Header `X-Api-Token` 取 token
  2. 用 token 查 source（走緩存）
  3. 驗證 employeeId + deviceId 屬於該 source（走緩存）
  4. 不匹配 → 403
  5. 通過 → 填入 sourceName, employeeId, deviceId
- `refresh-cache` 也改為 token 認證（移除 IP 驗證）

### 3c. 更新 `OrderSourceService.java` / `OrderSourceServiceImpl.java`
- 移除 `validateIp()` 方法
- 新增 `validateToken(String token)` → OmsOrderSource（帶 Redis 緩存）
- 新增 `validateDevice(Long sourceId, String employeeId, String deviceId)` → boolean（帶 Redis 緩存）
- Token 緩存 key: `auto_order:token:{token}` TTL 5min
- Device 緩存 key: `auto_order:devices:{sourceId}` TTL 5min（緩存整個 source 的設備列表）
- 移除 `IpUtil` 相關引用

### 3d. 更新 `AutoOrderServiceImpl.java`
- `generateCollectOrder`: order 填入 employeeId, deviceId；receiverName 用 employeeId
- `generatePayOrder`: 同上

### 3e. 安全白名單
- Portal 的 Spring Security 配置中，`/order/auto-generate/**` 路徑需要放行（它用自己的 token 驗證，不走 JWT）

---

## Step 4: Admin — 後台管理

### 4a. 更新 `OmsOrderSourceController.java`
- create 時自動生成 api_token（UUID 或 SecureRandom hex）
- 新增 `POST /orderSource/regenerateToken/{id}` — 重新生成 token 並清緩存
- Token 生成後回傳一次，之後查詢可選擇性遮罩顯示

### 4b. 新增 `OmsOrderSourceDeviceController.java`
- `GET /orderSource/{sourceId}/devices` — 列出該來源下的設備
- `POST /orderSource/{sourceId}/devices/create` — 新增設備
- `POST /orderSource/{sourceId}/devices/update/{id}` — 更新
- `POST /orderSource/{sourceId}/devices/delete/{id}` — 刪除

### 4c. 廠商角色 — 訂單過濾
- 修改 `OmsOrderController`（或相關訂單查詢 Service）
- 當前登入用戶有 sourceId 時，查詢自動加上 `source_name = ?` 過濾
- 廠商看不到其他來源的訂單
- 超級管理員（sourceId=null）看全部

### 4d. 廠商角色 — 菜單權限
- 廠商角色只能看：訂單列表、退貨列表（自己的）
- 不能看：商品管理、來源管理、系統設定等
- 在後台建一個「廠商」角色，配好菜單權限

---

## Step 5: 緩存策略

| 緩存 Key | 內容 | TTL | 清除時機 |
|----------|------|-----|---------|
| `auto_order:token:{token}` | OmsOrderSource | 5min | 重新生成 token 時 |
| `auto_order:devices:{sourceId}` | List<Device> | 5min | 設備增刪改時 |
| `auto_order:products` | List<Product> | 10min | 保留不變 |

---

## Step 6: 移除舊程式碼
- 刪除 `IpUtil.java`（如果沒有其他地方用）
- 移除 IP 白名單相關的所有緩存邏輯

---

## 驗收標準
1. `mvn clean package -DskipTests` 成功
2. curl 測試：
   - 無 token → 401/403
   - 錯誤 token → 403
   - 正確 token + 不屬於該來源的 employeeId → 403
   - 正確 token + 正確 employeeId + deviceId + amount → 200，訂單建立成功
3. 後台：
   - 來源管理可新增/編輯來源，顯示 token（可重新生成）
   - 設備管理可新增/刪除員工ID+設備號
   - 廠商帳號登入只看到自己的訂單
4. Docker 容器重啟成功，API 正常運作
