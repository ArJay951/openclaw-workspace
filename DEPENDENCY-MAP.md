# 前後端相依表 (Dependency Map)

改後端 API 時，查這張表確認要同步改哪些前端。
改前端時，確認後端 API 是否已就緒。

---

## General Mall（支付商城）

### 自定義 API（非原框架）

| 後端 API | Controller | 前端 API 檔 | 前端頁面 | 備註 |
|----------|-----------|-------------|---------|------|
| `/order/auto-generate/collect` | AutoOrderController (portal:8085) | — | 無前端（純 API） | 代收，X-Api-Token 認證 |
| `/order/auto-generate/pay` | AutoOrderController (portal:8085) | — | 無前端（純 API） | 代付 |
| `/order/auto-generate/refresh-cache` | AutoOrderController (portal:8085) | — | 無前端 | 緩存刷新 |
| `/orderSource/*` | OmsOrderSourceController (admin:8080) | `api/orderSource.js` | `views/oms/source/index.vue` | 來源管理+設備管理 |
| `/orderSource/{id}/devices/*` | OmsOrderSourceController (admin:8080) | `api/orderSource.js` | `views/oms/source/index.vue` | 設備 CRUD |
| `/orderSource/regenerateToken/{id}` | OmsOrderSourceController (admin:8080) | `api/orderSource.js` | `views/oms/source/index.vue` | Token 重新生成 |
| `/order/update/receiverInfo` | OmsOrderController (admin:8080) | `api/order.js` | `views/oms/order/orderDetail.vue` | 修改收件人 |
| `/returnApply/update/returnInfo` | OmsOrderReturnApplyController (admin:8080) | `api/returnApply.js` | `views/oms/apply/applyDetail.vue` | 修改退貨人 |

### 框架原有 API（改動風險低）

| 模組 | 後端 Controller | 前端 API 檔 | 前端頁面 |
|------|----------------|-------------|---------|
| 商品管理 | PmsProductController | `api/product.js` | `views/pms/product/` |
| 商品分類 | PmsProductCategoryController | `api/productCate.js` | `views/pms/productCate/` |
| 商品屬性 | PmsProductAttributeController | `api/productAttr.js` | `views/pms/productAttr/` |
| 品牌管理 | PmsBrandController | `api/brand.js` | `views/pms/brand/` |
| 訂單管理 | OmsOrderController | `api/order.js` | `views/oms/order/` |
| 退貨管理 | OmsOrderReturnApplyController | `api/returnApply.js` | `views/oms/apply/` |
| 退貨原因 | OmsOrderReturnReasonController | `api/returnReason.js` | `views/oms/reason/` |
| 優惠券 | SmsCouponController | `api/coupon.js` | `views/sms/coupon/` |
| 廣告管理 | SmsHomeAdvertiseController | `api/homeAdvertise.js` | `views/sms/advertise/` |
| 秒殺活動 | SmsFlashPromotionController | `api/flash.js` | `views/sms/flash/` |
| 用戶管理 | UmsAdminController | `api/login.js` | `views/ums/admin/` |
| 角色管理 | UmsRoleController | `api/role.js` | `views/ums/role/` |
| 菜單管理 | UmsMenuController | `api/menu.js` | `views/ums/menu/` |
| 資源管理 | UmsResourceController | `api/resource.js` | `views/ums/resource/` |

### 路徑對照
- 後端: `/home/ubuntu/general-mall/mall/`
- Admin 前端: `/home/ubuntu/mall-admin-web/`
- 測試腳本: `/home/ubuntu/general-mall/tests/`

---

## Entertainment Mall（娛樂城商城）

### 自定義 API（SaaS 多租戶）

| 後端 API | Controller | 前端 API 檔 | 前端頁面 | 備註 |
|----------|-----------|-------------|---------|------|
| `/tenant/info` (portal) | TenantController (portal:8095) | `api/tenant.js` (app) | `App.vue` (app) | 租戶品牌資訊 |
| `/tenant/info` (admin) | TenantController (admin:8090) | — | `App.vue` (admin) | 後台租戶名稱 |

### 前台 App（Vue 2 + Vant 2）

| 功能 | 前端 API 檔 | 前端頁面 | 後端 Controller (portal:8095) |
|------|-------------|---------|------------------------------|
| 首頁 | `api/home.js` | `views/home/index.vue` | HomeController |
| 商品搜尋/列表 | `api/product.js` | `views/product/list.vue` | PortalProductController |
| 商品詳情 | `api/product.js` | `views/product/detail.vue` | PortalProductController |
| 分類 | `api/product.js` | `views/category/index.vue` | PortalProductController |
| 購物車 | `api/cart.js` | `views/cart/index.vue` | CartItemController |
| 訂單 | `api/order.js` | `views/order/` | OmsPortalOrderController |
| 收藏 | `api/collection.js` | — | MemberCollectionController |
| 收貨地址 | `api/address.js` | `views/address/` | MemberReceiveAddressController |
| 會員 | `api/member.js` | `views/user/`, `views/login/` | UmsMemberController |
| 優惠券 | `api/coupon.js` | `views/coupon/` | CouponController |

### 後台 Admin（Vue 2 + Element UI）
與 General Mall 結構相同（同源 macrozheng/mall），主要差異：
- 金額顯示「點」不是「元」
- 多了 `/tenant/info` 動態載入租戶名稱

| 模組 | 前端 API 檔 | 前端頁面 | 備註 |
|------|-------------|---------|------|
| 商品管理 | `api/product.js` | `views/pms/product/` | 價格顯示「點」 |
| 訂單管理 | `api/order.js` | `views/oms/order/` | 金額顯示「點」 |
| 退貨管理 | `api/returnApply.js` | `views/oms/apply/` | |
| 優惠券 | `api/coupon.js` | `views/sms/coupon/` | 面額顯示「點」 |

### 路徑對照
- 後端: `/home/ubuntu/entertainment-mall/`
- Admin 前端: `/home/ubuntu/ent-mall-admin-web/`
- App 前端: `/home/ubuntu/ent-mall-app-web/`
- 測試腳本: `/home/ubuntu/entertainment-mall/tests/`

---

## 🔔 開工單檢查規則

改後端時，必須確認：
1. ☐ 前端 API 檔是否需要新增/修改呼叫
2. ☐ 前端頁面是否需要同步更新 UI
3. ☐ 測試腳本是否需要新增/更新 test case
4. ☐ 如果改了 Model 欄位，Mapper + 前端顯示都要跟

改前端時，必須確認：
1. ☐ 後端 API 是否已存在且就緒
2. ☐ API 回傳格式是否與前端預期一致
3. ☐ build 後部署到正確路徑

---

*Last updated: 2026-02-13*
