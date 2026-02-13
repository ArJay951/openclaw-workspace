# 待移除模組清單

## 優惠促銷模組（全部移除）

### 資料庫表
```sql
-- 優惠券
DROP TABLE IF EXISTS sms_coupon;
DROP TABLE IF EXISTS sms_coupon_history;
DROP TABLE IF EXISTS sms_coupon_product_category_relation;
DROP TABLE IF EXISTS sms_coupon_product_relation;

-- 限時促銷
DROP TABLE IF EXISTS sms_flash_promotion;
DROP TABLE IF EXISTS sms_flash_promotion_log;
DROP TABLE IF EXISTS sms_flash_promotion_product_relation;
DROP TABLE IF EXISTS sms_flash_promotion_session;

-- 商品優惠
DROP TABLE IF EXISTS pms_product_full_reduction;  -- 滿減
DROP TABLE IF EXISTS pms_product_ladder;          -- 階梯價
DROP TABLE IF EXISTS pms_member_price;            -- 會員價
```

### 後端代碼
- `mall-admin/src/main/java/com/macro/mall/controller/SmsCoupon*`
- `mall-admin/src/main/java/com/macro/mall/controller/SmsFlashPromotion*`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPromotionServiceImpl.java`

### 前端頁面
- `src/views/sms/coupon/` - 優惠券管理
- `src/views/sms/flash/` - 限時促銷
- 商品編輯頁面中的「促銷設置」區塊

### 後台菜單
移除 `ums_menu` 中的：
- 營銷 > 優惠券列表
- 營銷 > 品牌推薦（可選保留）
- 營銷 > 新品推薦（可選保留）
- 營銷 > 人氣推薦（可選保留）
- 營銷 > 專題推薦（可選保留）
- 營銷 > 廣告列表（可選保留）
- 營銷 > 秒殺活動

---

## 支付模組（改造）

### 移除
- 支付寶支付
- 微信支付

### 新增
- 點數支付（對接娛樂城 API）

---

## 執行策略

**Phase 1**：先保留表結構，前端隱藏入口
**Phase 2**：後端移除相關 API
**Phase 3**：完全移除表和代碼（可選）

> 建議先做 Phase 1，快速上線後再慢慢清理
