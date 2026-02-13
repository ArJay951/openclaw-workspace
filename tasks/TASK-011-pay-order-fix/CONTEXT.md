# CONTEXT.md — TASK-011

## 路徑
- 後端: `/home/ubuntu/general-mall/mall/`
- 核心檔案: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java`
- 回傳物件: `mall-portal/src/main/java/com/macro/mall/portal/domain/AutoOrderResult.java`
- 測試腳本: `/home/ubuntu/general-mall/tests/`

## Docker
- Build: `cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests`
- Portal 容器重啟方式: 查 docker-compose 或手動 `docker restart`

## 測試用資料
- API Token (claw): 36ae430b6120992e5ac779cd8713342e
- 測試員工: employeeId=emp001, deviceId=dev001
- Portal port: 8085
