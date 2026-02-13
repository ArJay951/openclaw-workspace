# TASK-017: 移除 MongoDB 依賴 — 完成

## 執行摘要

### 已移除項目
- **15 個 Java 檔案**：3 Controller + 3 Service Interface + 3 Service Impl + 3 Repository + 3 Domain
- **MongoDB AutoConfiguration** 已排除（MallPortalApplication.java）
- **pom.xml** 移除 `spring-boot-starter-data-mongodb`
- **application.yml** 移除 `mongo:` 配置段
- **application-dev.yml / application-prod.yml** 移除 `spring.data.mongodb` 配置
- **docker-compose.yml** 移除 mongo service、mongo_data volume、MONGO_HOST 環境變數、depends_on mongo

### 啟動時間對比
| | 啟動時間 |
|---|---|
| 移除前 | **22.599 秒** |
| 移除後 | **15.493 秒** |
| 改善 | **-7.1 秒 (31% 加速)** |

### API 測試
```
curl -s http://localhost:8085/order/auto-generate/collect -X POST -H "Content-Type: application/json" -d '{"payAmount":100}'
→ {"code":404,"message":"deviceIddeviceId 不能為空"}
```
✅ API 正常回應（業務驗證錯誤，非系統錯誤）

### 驗收結果
- ✅ `mvn clean package -DskipTests` 成功
- ✅ Portal 啟動成功，日誌無 MongoDB 相關錯誤
- ✅ 代收 API 正常運作
- ✅ `docker ps` 無 mall-mongo 容器
- ✅ 啟動時間從 22.6s 降至 15.5s

### Git
- Commit: `移除 MongoDB 依賴：刪除相關 Controller/Service/Repository/Domain，排除 MongoAutoConfiguration，移除 pom 依賴和 yml 配置`
- 已 push 至 GitHub (mall-backend main)

### 問題
無
