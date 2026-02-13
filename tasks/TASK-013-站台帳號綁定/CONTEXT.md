# CONTEXT - TASK-013

## 專案路徑
- 後端: `/home/ubuntu/general-mall/mall/`
- 前端: `/home/ubuntu/mall-admin-web/`

## 關鍵檔案

### 後端
- `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderSourceController.java` — 站台 CRUD API
- `mall-admin/src/main/java/com/macro/mall/service/OmsOrderSourceService.java` — Service 介面
- `mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderSourceServiceImpl.java` — Service 實作（改這裡）
- `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java` — 已有 applySourceFilter，參考用
- `mall-admin/src/main/java/com/macro/mall/controller/OmsOrderReturnApplyController.java` — 退貨申請（需加 source 過濾）
- `mall-admin/src/main/java/com/macro/mall/dto/OmsReturnApplyQueryParam.java` — 已有 sourceName 欄位
- `mall-admin/src/main/resources/dao/OmsOrderReturnApplyDao.xml` — SQL 已有 sourceName 過濾條件（line 25, 49-50）
- `mall-admin/src/main/java/com/macro/mall/service/UmsAdminService.java` — 帳號管理
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java` — register() 可參考

### 前端
- `src/views/oms/source/index.vue` — 站台管理頁（改這裡）

## DB Schema

### ums_admin（帳號表）
| Field | Type | Note |
|-------|------|------|
| id | bigint PK | |
| username | varchar(64) | 唯一 |
| password | varchar(64) | BCrypt |
| source_id | bigint | **已存在**，綁定站台 |
| status | int | 1=啟用 |
| nick_name | varchar(200) | |
| create_time | datetime | |

### ums_role（角色表）
| id | name |
|----|------|
| 1 | 超级管理员 |
| (新) | 站台角色 |

### ums_resource（權限資源）
| id | name | url |
|----|------|-----|
| 7 | 訂單管理 | /order/** |
| 8 | 訂單退貨申請管理 | /returnApply/** |

### 關聯表
- `ums_admin_role_relation`: admin_id + role_id
- `ums_role_resource_relation`: role_id + resource_id

### oms_order_source（站台表）
- id, name, api_token, status, device_count, create_time
- **沒有 admin_id 欄位**（帳號透過 ums_admin.source_id 反查）

### oms_order_return_apply（退貨申請表）
- **沒有 source_name 欄位**
- XML 裡是透過 JOIN oms_order 的 source_name 過濾
- ⚠️ 代付訂單 orderId=0，JOIN 不到 order → 需要處理
- 方案A: 在 return_apply 表加 source_name 欄位（代付時寫入）
- 方案B: 在代付 API (OmsPortalOrderController.generatePayOrder) 建 return_apply 時帶入 source_name
- **建議用方案A+B**: 加欄位 + 代付時寫入 + XML 過濾改為直接讀 return_apply.source_name

## 代付 API 位置
- `mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java` — generatePayOrder()
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java` — generatePayOrder 實作

## 已有的 Source 過濾邏輯（OmsOrderController 參考）
```java
private void applySourceFilter(OmsOrderQueryParam queryParam) {
    Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    if (principal instanceof UserDetails) {
        String username = ((UserDetails) principal).getUsername();
        UmsAdmin admin = adminService.getAdminByUsername(username);
        if (admin != null && admin.getSourceId() != null) {
            OmsOrderSource source = orderSourceMapper.selectByPrimaryKey(admin.getSourceId());
            if (source != null) {
                queryParam.setSourceName(source.getName());
            }
        }
    }
}
```

## Build & Deploy
```bash
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests
# 如需重建 Docker image:
cd /home/ubuntu/general-mall && docker compose build mall-admin mall-portal && docker compose up -d
```

## Test
```bash
cd /home/ubuntu/general-mall && bash tests/run-tests.sh
```

## 技術棧
- Spring Boot 2.7, MyBatis, Spring Security
- Vue 2 + Element UI
- MySQL 5.7, Redis
- BCrypt 密碼加密: `new BCryptPasswordEncoder().encode(rawPassword)`
- 前端動態選單由 role → resource → menu 控制，不需額外配置

## Ports
- mall-admin: 8080
- mall-portal: 8085
