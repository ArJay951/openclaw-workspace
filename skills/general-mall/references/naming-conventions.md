# 命名規範

## 資料庫欄位名稱

### ⚠️ 常見拼寫錯誤

| ❌ 錯誤 | ✅ 正確 | 說明 |
|---------|---------|------|
| `feight_template_id` | `freight_template_id` | 運費模板 ID |
| `recommand_status` | `recommend_status` | 推薦狀態 |

### 正確欄位列表 (pms_product)

```sql
-- 商品表重要欄位
id
brand_id
product_category_id
freight_template_id          -- ✓ 運費模板
product_attribute_category_id
name
pic
product_sn
delete_status
publish_status
new_status
recommend_status             -- ✓ 推薦狀態
verify_status
sort
sale
price
promotion_price
gift_point
gift_growth
use_point_limit
sub_title
original_price
stock
low_stock
unit
weight
preview_status
service_ids
keywords
note
album_pics
detail_title
detail_desc
detail_html
detail_mobile_html
promotion_start_time
promotion_end_time
promotion_per_limit
promotion_type
brand_name
product_category_name
```

## Java 命名規範

- 類名: PascalCase (`FreightTemplate`)
- 方法名: camelCase (`getFreightTemplate`)
- 常量: UPPER_SNAKE_CASE (`DEFAULT_FREIGHT_ID`)
- 包名: lowercase (`com.macro.mall.portal`)

## API 命名規範

- RESTful 風格
- 資源名用複數 (`/products`, `/orders`)
- 動作用 HTTP 方法 (`GET`, `POST`, `PUT`, `DELETE`)

```
GET    /api/products          # 列表
GET    /api/products/{id}     # 詳情
POST   /api/products          # 新增
PUT    /api/products/{id}     # 更新
DELETE /api/products/{id}     # 刪除
```
