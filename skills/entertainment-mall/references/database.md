# 数据库设计文档

## 概述

基于 mall 的数据库设计，共 75 张表，分为 5 个模块。

源文件: `/home/ubuntu/mall-source/document/sql/mall.sql`

## 模块划分

### CMS - 内容管理 (12 张表)

| 表名 | 说明 |
|------|------|
| cms_help | 帮助文档 |
| cms_help_category | 帮助分类 |
| cms_member_report | 用户举报 |
| cms_prefrence_area | 优选专区 |
| cms_prefrence_area_product_relation | 优选专区商品关联 |
| cms_subject | 专题 |
| cms_subject_category | 专题分类 |
| cms_subject_comment | 专题评论 |
| cms_subject_product_relation | 专题商品关联 |
| cms_topic | 话题 |
| cms_topic_category | 话题分类 |
| cms_topic_comment | 话题评论 |

### OMS - 订单管理 (8 张表)

| 表名 | 说明 |
|------|------|
| oms_cart_item | 购物车 |
| oms_company_address | 公司地址（退货用）|
| oms_order | 订单主表 |
| oms_order_item | 订单商品 |
| oms_order_operate_history | 订单操作记录 |
| oms_order_return_apply | 退货申请 |
| oms_order_return_reason | 退货原因 |
| oms_order_setting | 订单设置 |

### PMS - 商品管理 (18 张表)

| 表名 | 说明 |
|------|------|
| pms_album | 相册 |
| pms_album_pic | 相册图片 |
| pms_brand | 品牌 |
| pms_comment | 商品评论 |
| pms_comment_replay | 评论回复 |
| pms_feight_template | 运费模板 |
| pms_member_price | 会员价格 |
| pms_product | 商品主表 |
| pms_product_attribute | 商品属性 |
| pms_product_attribute_category | 属性分类 |
| pms_product_attribute_value | 属性值 |
| pms_product_category | 商品分类 |
| pms_product_category_attribute_relation | 分类属性关联 |
| pms_product_full_reduction | 满减规则 |
| pms_product_ladder | 阶梯价格 |
| pms_product_operate_log | 商品操作日志 |
| pms_product_vertify_record | 商品审核记录 |
| pms_sku_stock | SKU 库存 |

### SMS - 促销管理 (12 张表)

| 表名 | 说明 |
|------|------|
| sms_coupon | 优惠券 |
| sms_coupon_history | 优惠券领取记录 |
| sms_coupon_product_category_relation | 优惠券分类关联 |
| sms_coupon_product_relation | 优惠券商品关联 |
| sms_flash_promotion | 限时购活动 |
| sms_flash_promotion_log | 限时购日志 |
| sms_flash_promotion_product_relation | 限时购商品关联 |
| sms_flash_promotion_session | 限时购场次 |
| sms_home_advertise | 首页广告 |
| sms_home_brand | 首页品牌推荐 |
| sms_home_new_product | 首页新品推荐 |
| sms_home_recommend_product | 首页人气推荐 |
| sms_home_recommend_subject | 首页专题推荐 |

### UMS - 用户管理 (25 张表)

| 表名 | 说明 |
|------|------|
| ums_admin | 后台管理员 |
| ums_admin_login_log | 管理员登录日志 |
| ums_admin_permission_relation | 管理员权限关联 |
| ums_admin_role_relation | 管理员角色关联 |
| ums_growth_change_history | 成长值变更记录 |
| ums_integration_change_history | 积分变更记录 |
| ums_integration_consume_setting | 积分消费设置 |
| ums_member | 会员 |
| ums_member_level | 会员等级 |
| ums_member_login_log | 会员登录日志 |
| ums_member_member_tag_relation | 会员标签关联 |
| ums_member_product_category_relation | 会员分类关联 |
| ums_member_receive_address | 收货地址 |
| ums_member_rule_setting | 会员规则设置 |
| ums_member_statistics_info | 会员统计信息 |
| ums_member_tag | 会员标签 |
| ums_member_task | 会员任务 |
| ums_menu | 菜单 |
| ums_permission | 权限 |
| ums_resource | 资源 |
| ums_resource_category | 资源分类 |
| ums_role | 角色 |
| ums_role_menu_relation | 角色菜单关联 |
| ums_role_permission_relation | 角色权限关联 |
| ums_role_resource_relation | 角色资源关联 |

## 需要扩展的表

### 娱乐城对接

```sql
-- 新增：娱乐城用户映射表
CREATE TABLE ums_casino_member_mapping (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT NOT NULL COMMENT '商城会员ID',
  casino_user_id VARCHAR(100) NOT NULL COMMENT '娱乐城用户ID',
  casino_code VARCHAR(50) NOT NULL COMMENT '娱乐城代码',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY (casino_code, casino_user_id)
);

-- 新增：点数交易记录表
CREATE TABLE oms_points_transaction (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT NOT NULL,
  order_id BIGINT COMMENT '关联订单',
  transaction_type TINYINT COMMENT '1-扣除 2-退还',
  points DECIMAL(10,2) NOT NULL,
  casino_transaction_id VARCHAR(100) COMMENT '娱乐城交易ID',
  status TINYINT COMMENT '0-待处理 1-成功 2-失败',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 新增：虚拟商品发放记录表
CREATE TABLE oms_virtual_delivery (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT NOT NULL,
  order_item_id BIGINT NOT NULL,
  delivery_type TINYINT COMMENT '1-彩金 2-虚拟卡',
  amount DECIMAL(10,2) COMMENT '发放金额',
  casino_transaction_id VARCHAR(100),
  status TINYINT COMMENT '0-待发放 1-成功 2-失败',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 多租户配置

```sql
-- 新增：商城配置表
CREATE TABLE sys_mall_config (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  config_key VARCHAR(100) NOT NULL UNIQUE,
  config_value TEXT,
  description VARCHAR(255),
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 初始配置
INSERT INTO sys_mall_config (config_key, config_value, description) VALUES
('mall_name', '精品商城', '商城名称'),
('mall_logo', '', '商城 Logo URL'),
('casino_api_url', '', '娱乐城 API 地址'),
('casino_api_key', '', '娱乐城 API 密钥');
```
