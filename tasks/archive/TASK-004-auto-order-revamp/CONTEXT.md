# CONTEXT.md — TASK-004

## 專案路徑
- **後端根目錄**: `/home/ubuntu/general-mall/mall/`
- **模組結構**:
  - `mall-mbg/` — MyBatis Generator, Model, Mapper
  - `mall-portal/` — 前台 API（Auto-Order 在這）
  - `mall-admin/` — 後台管理 API
  - `mall-common/` — 共用工具
  - `mall-security/` — Spring Security 配置

## 關鍵檔案

### Portal（改 API 認證）
- `mall-portal/src/main/java/com/macro/mall/portal/controller/AutoOrderController.java`
- `mall-portal/src/main/java/com/macro/mall/portal/domain/AutoOrderRequest.java`
- `mall-portal/src/main/java/com/macro/mall/portal/domain/AutoOrderResult.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/AutoOrderService.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/OrderSourceService.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OrderSourceServiceImpl.java`
- `mall-portal/src/main/java/com/macro/mall/portal/util/IpUtil.java`
- Security 白名單: `mall-portal/src/main/java/com/macro/mall/portal/config/` (找 SecurityConfig 或 MallSecurityConfig)

### Admin（改後台管理）
- `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderSourceController.java`
- `mall-admin/src/main/java/com/macro/mall/service/OmsOrderSourceService.java`
- `mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderSourceServiceImpl.java`
- 訂單相關 Controller: `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java`

### MBG（Model + Mapper）
- `mall-mbg/src/main/java/com/macro/mall/model/OmsOrderSource.java`
- `mall-mbg/src/main/java/com/macro/mall/model/OmsOrder.java`
- `mall-mbg/src/main/java/com/macro/mall/model/UmsAdmin.java`
- `mall-mbg/src/main/java/com/macro/mall/mapper/OmsOrderSourceMapper.java`

## 現有 DB（oms_order_source）
```
id | name     | ip_whitelist                        | status | create_time         
1  | 測試來源  | 127.0.0.1,172.17.0.0/16,172.18.0.0/16 | 1   | 2026-02-11 06:45:24
2  | claw     | 52.76.231.27                         | 1      | 2026-02-11 07:40:49
3  | jay_home | 61.231.54.52                         | 1      | 2026-02-11 07:47:40
```

## OmsOrder 來源相關欄位
- `source_type` INT(1)
- `source_domain` VARCHAR(200)
- `source_ip` VARCHAR(45)
- `source_name` VARCHAR(200)

## UmsAdmin 現有欄位
- id, username, password, icon, email, nickName, note, createTime, loginTime, status
- 無 sourceId（需新增）

## Docker
- MySQL 容器名: `mall-mysql`
- 連線: `docker exec mall-mysql mysql -u root -proot mall`
- 後端 build: `cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests`
- Admin 容器: 用 docker-compose（查 `/home/ubuntu/general-mall/` 或 `/home/ubuntu/mall-deploy/`）

## 技術棧
- Java 8 + Spring Boot 2.7
- MyBatis（annotation-based mapper，非 XML）
- Redis 緩存（`RedisService` in mall-common）
- Swagger API 文件

## ⚠️ 注意
- Mapper 用 annotation（@Select, @Insert 等），不是 XML
- `OmsOrderSource` 的序列化要注意 Redis 緩存相容性
- Token 不要用 JWT，用簡單的 random hex string（64 char），存 DB
- 廠商角色的訂單過濾，查看現有 OmsOrderController 的查詢方式再決定怎麼加 where 條件
