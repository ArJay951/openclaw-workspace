# TASK-004: DONE

## 完成摘要

### ✅ 已完成項目

#### Step 1: DB 改造
- `oms_order_source`: 新增 `api_token` 欄位，移除 `ip_whitelist`
- 新增 `oms_order_source_device` 表（員工+設備對應）
- `oms_order`: 新增 `employee_id`, `device_id` 欄位
- `ums_admin`: 新增 `source_id` 欄位
- 已為現有 3 個來源生成 Token

#### Step 2: Model 層 (mall-mbg)
- `OmsOrderSource.java`: 移除 ipWhitelist，新增 apiToken
- 新增 `OmsOrderSourceDevice.java` Model
- 新增 `OmsOrderSourceDeviceMapper.java` (annotation-based)
- `OmsOrderSourceMapper.java`: 新增 selectByApiToken、regenerateToken
- `OmsOrder.java`: 新增 employeeId, deviceId
- `UmsAdmin.java`: 新增 sourceId
- 更新對應的 MyBatis XML 映射（OmsOrderMapper.xml, UmsAdminMapper.xml）

#### Step 3: Portal — API 認證改造
- `AutoOrderRequest.java`: 移除 name/phone，新增 employeeId/deviceId
- `AutoOrderController.java`: IP 白名單 → Token 認證 (`X-Api-Token` header) + 設備驗證
- `OrderSourceService.java`: 移除 validateIp，新增 validateToken + validateDevice
- `OrderSourceServiceImpl.java`: Redis 緩存（token 5min, devices 5min）
- `AutoOrderServiceImpl.java`: order 填入 employeeId/deviceId, receiverName 用 employeeId
- 刪除 `IpUtil.java`
- `/order/auto-generate/**` 已在 Security 白名單中（原本就有）

#### Step 4: Admin — 後台管理
- `OmsOrderSourceController.java`: 新增 regenerateToken、設備 CRUD API
- `OmsOrderSourceService.java` / `Impl`: 新增 regenerateToken、設備管理、自動生成 Token
- `OmsOrderController.java`: 廠商角色過濾（sourceId → sourceName 過濾訂單）

#### Step 6: 移除舊程式碼
- 刪除 `IpUtil.java`
- 移除所有 IP 白名單相關邏輯

### 驗收結果

#### ✅ mvn clean package -DskipTests → BUILD SUCCESS
所有 8 個模組編譯通過。

#### ✅ Docker 容器重啟成功
mall-portal 和 mall-admin 已更新 jar 並重啟。

#### ✅ curl 測試
| 測試 | 預期 | 結果 |
|------|------|------|
| 無 Token | 401/403 | ✅ code=401 "暂未登录" |
| 錯誤 Token | 403 | ✅ code=403 "没有相关权限" |
| 正確 Token + 不屬於該來源的 employee | 403 | ✅ code=403 |
| 正確 Token + 正確 employee + deviceId | 200 | ✅ 訂單建立成功 |
| Pay order 測試 | 200 | ✅ 退貨訂單建立成功 |
| DB 驗證 employee_id/device_id 寫入 | 有值 | ✅ 確認寫入 |

### API Token
| 來源 | Token |
|------|-------|
| 測試來源 | 550711ffb944efb498366a45940f736e |
| claw | 36ae430b6120992e5ac779cd8713342e |
| jay_home | a2ba0a5956d918ce91c5732134f8f8a0 |

### ⚠️ 備註
- Step 4d（廠商角色菜單權限配置）需在後台手動建立「廠商」角色並配好菜單權限，這是後台 UI 操作不是程式碼改動
- 緩存清除：clearCache() 依賴 TTL 自動過期（5min），重新生成 Token 時會清除對應 token 緩存
- 測試設備已建立：source_id=2(claw), employeeId=emp001, deviceId=dev001
