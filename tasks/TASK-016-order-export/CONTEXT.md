# CONTEXT - TASK-016

## 路徑
- 後端: `/home/ubuntu/general-mall/mall/`
  - Controller: `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java`
  - QueryParam: `mall-admin/src/main/java/com/macro/mall/dto/OmsOrderQueryParam.java`
  - Service Interface: `mall-admin/src/main/java/com/macro/mall/service/OmsOrderService.java`
  - Service Impl: `mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderServiceImpl.java`
  - DAO XML: `mall-admin/src/main/resources/dao/OmsOrderDao.xml`
  - OmsOrderDao: `mall-admin/src/main/java/com/macro/mall/dao/OmsOrderDao.java`
  - OmsOrderItemMapper: `mall-mbg/src/main/java/com/macro/mall/mapper/OmsOrderItemMapper.java`
- 前端: `/home/ubuntu/mall-admin-web/`
  - 訂單列表: `src/views/oms/order/index.vue`
  - API: `src/api/order.js`
  - request util: `src/utils/request.js`

## DB
- MySQL: `docker exec -i mall-mysql mysql -u root -proot mall`
- 表: `oms_order`, `oms_order_item`

## 現有 OmsOrderQueryParam 欄位
orderSn, receiverKeyword, status, orderType, sourceType, createTime, sourceName

## 現有 OmsOrderDao.xml getList
- createTime 用 LIKE (只支援單日)
- 有 sourceName 過濾

## Build
```bash
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-admin/target && docker build -t mall/mall-admin:1.0-SNAPSHOT -f /tmp/Dockerfile-admin .
cd /home/ubuntu/general-mall && docker compose up -d mall-admin
cd /home/ubuntu/mall-admin-web && npm run build
```

## 前端 request.js 注意事項
- 可能有 response interceptor 對所有 response 做 `response.data` 處理
- blob 下載需要特殊處理：要麼在 interceptor 裡判斷 responseType，要麼用原生 XMLHttpRequest
- 如果 interceptor 有問題，最簡單方案：直接用 window.open + 後端支援 token query param

## hutool 依賴檢查
```bash
grep -r "hutool\|poi" /home/ubuntu/general-mall/mall/mall-admin/pom.xml
grep -r "hutool\|poi" /home/ubuntu/general-mall/mall/pom.xml
```
項目使用 hutool，如果已有 hutool-all 則不需額外加依賴。
