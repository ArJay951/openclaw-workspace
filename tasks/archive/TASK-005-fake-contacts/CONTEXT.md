# CONTEXT.md — TASK-005

## 專案路徑
- **後端根目錄**: `/home/ubuntu/general-mall/mall/`
- `mall-mbg/` — Model, Mapper
- `mall-portal/` — Auto-Order API
- `mall-admin/` — 後台管理

## 關鍵檔案
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java` — 代收代付核心邏輯
- `mall-portal/src/main/java/com/macro/mall/portal/controller/AutoOrderController.java`
- `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java` — 訂單管理
- `mall-mbg/src/main/java/com/macro/mall/model/` — Model 目錄
- `mall-mbg/src/main/java/com/macro/mall/mapper/` — Mapper 目錄

## 目前 AutoOrderServiceImpl 中的姓名/電話邏輯
- `generateCollectOrder`: `order.setReceiverName(request.getName())` → 但 TASK-004 已改為用 employeeId
- `generatePayOrder`: 同上，退貨申請也填 name/phone
- TASK-004 把 name/phone 從 request 移除了，現在 receiverName 用的是 employeeId

## Docker
- MySQL 容器: `mall-mysql`
- 連線: `docker exec mall-mysql mysql -u root -proot mall`
- Build: `cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests`

## 技術棧
- Java 8 + Spring Boot 2.7 + MyBatis (annotation-based mapper)
- Redis 緩存 (RedisService in mall-common)

## Auto-Order API Token 測試用
- Token (claw): `36ae430b6120992e5ac779cd8713342e`
- 測試設備: employeeId=emp001, deviceId=dev001

## ⚠️ 注意
- Mapper 用 annotation (@Select 等)，不是 XML
- 假人資料用腳本直接 INSERT 到 MySQL 即可（不用 Java 程式生成）
- 台灣手機格式: 09 + 8 位數字（共 10 位）
