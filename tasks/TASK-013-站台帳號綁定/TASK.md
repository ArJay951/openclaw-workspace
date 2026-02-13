# TASK-013: 站台帳號自動建立 + 權限隔離

## 目標
新增站台時自動建立一個後台帳號，該帳號只能**查看**訂單列表和退貨申請列表，且只能看到**自己站台**的資料。

## 需求
1. **建立站台時自動建帳號**: OmsOrderSourceServiceImpl.create() 中，建完 source 後自動建立 ums_admin（username=站台名稱, password=隨機8位, source_id=新站台id）
2. **建立「站台」角色**: 插入 ums_role（name=站台角色），分配 resource 只有 id=7（訂單管理）和 id=8（退貨申請管理）
3. **新帳號自動綁定角色**: 建帳號時同時插入 ums_admin_role_relation
4. **退貨申請列表加 source 過濾**: OmsOrderReturnApplyController.list() 加上與 OmsOrderController 相同的 applySourceFilter 邏輯
5. **前端：站台列表顯示帳號/密碼**: 新增站台成功後，彈窗顯示自動生成的帳號密碼（密碼只顯示一次）
6. **站台帳號只能看，不能操作**: 權限 resource 設為只讀（GET 請求）

## 步驟

### 後端

#### 1. 建「站台」角色（如果不存在）
- 在 `OmsOrderSourceServiceImpl` 的 `create()` 方法中：
  - 檢查 ums_role 是否已有 name='站台角色' 的記錄，沒有就 INSERT
  - 確保 ums_role_resource_relation 綁定 resource_id = 7, 8

#### 2. create() 自動建帳號
```java
// 在 OmsOrderSourceServiceImpl.create() 中
// 1. 建 source（已有）
// 2. 生成帳號
String username = source.getName(); // 站台名稱當帳號
String rawPassword = generateRandomPassword(8); // 8位隨機密碼
// 3. 呼叫 UmsAdminService.register() 或直接 insert
UmsAdmin admin = new UmsAdmin();
admin.setUsername(username);
admin.setPassword(passwordEncoder.encode(rawPassword));
admin.setSourceId(source.getId());
admin.setStatus(1);
admin.setCreateTime(new Date());
admin.setNickName(source.getName());
// 4. 綁定「站台角色」
// 5. 回傳時把 rawPassword 帶回（只顯示一次）
```

#### 3. 退貨申請加 source 過濾
- `OmsOrderReturnApplyController` 注入 `UmsAdminService` 和 `OmsOrderSourceMapper`
- 在 `list()` 方法裡取得當前用戶，若有 sourceId 則設定 `queryParam.setSourceName(source.getName())`
- XML 裡已有 sourceName 過濾條件，不需改

#### 4. create() 回傳新增帳號資訊
- 新增 DTO `OmsOrderSourceCreateResult`，包含 source + username + rawPassword
- create API 回傳改為此 DTO

### 前端（mall-admin-web）

#### 5. 站台管理頁 (source/index.vue)
- 新增站台成功後，彈窗顯示帳號密碼（類似 Token 顯示的對話框）
- 密碼旁加「複製」按鈕
- 列表新增「綁定帳號」欄位顯示 username

#### 6. 站台帳號登入後前端選單限制
- 這部分不用改，因為 mall 的選單權限是由 role 的 resource 控制的
- 後台的動態路由機制會自動只顯示有權限的選單

## 驗收標準
1. 新增站台 → 自動建帳號 → 彈窗顯示帳號密碼 ✓
2. 用站台帳號登入 → 只看到訂單列表和退貨申請 ✓
3. 訂單列表只顯示該站台的訂單 ✓
4. 退貨申請列表只顯示該站台的退貨 ✓
5. 站台帳號無法存取其他功能（商品、行銷等）✓
6. 原 admin 帳號不受影響 ✓
7. build 成功，67 tests 全 PASS ✓

## ⚠️ 注意
- 密碼用 BCrypt 加密存儲，明文只在建立時回傳一次
- 站台名稱不能重複（同時也是帳號名，ums_admin.username 有唯一約束）
- 刪除站台時要一併刪除帳號和角色關聯
- `OmsOrderReturnApplyDao.xml` 的 source 過濾是透過 JOIN oms_order 的 source_name 欄位
- 代付訂單（orderId=0）沒有對應 order，需要特殊處理：oms_order_return_apply 表可能需要加 source_name 欄位
