# 自動產生訂單 API

## 功能說明

提供 API 讓下游系統傳入金額、姓名、手機號，自動根據後台商品產生符合總額的訂單。

## API 規格

### 請求

```
POST /api/order/auto-generate
```

```json
{
  "amount": 117,
  "name": "張三",
  "phone": "0912345678"
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| amount | number | ✅ | 訂單金額 |
| name | string | ✅ | 收件人姓名 |
| phone | string | ✅ | 收件人手機 |

### 成功回應

```json
{
  "success": true,
  "order_id": "202602060001",
  "items": [
    {"product_id": 3, "name": "100元商品", "price": 100, "qty": 1},
    {"product_id": 1, "name": "20元商品", "price": 20, "qty": 1}
  ],
  "subtotal": 120,
  "discount": 3,
  "total": 117
}
```

### 錯誤回應

```json
{
  "success": false,
  "error_code": "AMOUNT_TOO_LOW",
  "error_message": "金額不足，最低需要 20 元"
}
```

## 處理邏輯

### 商品選擇演算法

1. 取得所有上架商品，按價格降序排列
2. 從最高價商品開始，盡可能填滿金額
3. 同一商品可選多件
4. 剩餘金額若低於最低價商品 → 產生折扣券

### 演算法示例

```
商品：[100, 50, 20]（價格降序）
傳入金額：117

Step 1: 117 ÷ 100 = 1 餘 17 → 選 100×1
Step 2: 17 < 50，跳過
Step 3: 17 ÷ 20 = 0 餘 17 → 選 20×0（不足）
Step 4: 17 < 20（最低價）→ 產生折扣

調整：回退，嘗試多選一件低價商品
選 100×1 + 20×1 = 120
折扣 = 120 - 117 = 3（< 20，符合規則）

結果：100×1 + 20×1，折扣 3 元
```

### 折扣規則

- 折扣金額 ≤ 最低價商品金額
- 折扣以「折價券」形式記錄

### 錯誤處理

| 情況 | 處理 |
|------|------|
| 金額 < 最低價商品 | 返回 AMOUNT_TOO_LOW 錯誤 |
| 無上架商品 | 返回 NO_PRODUCTS 錯誤 |
| 參數缺失 | 返回 INVALID_PARAMS 錯誤 |

## 資料庫擴展

```sql
-- 自動訂單記錄表
CREATE TABLE oms_auto_order_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT NOT NULL,
  request_amount DECIMAL(10,2) NOT NULL,
  actual_amount DECIMAL(10,2) NOT NULL,
  discount_amount DECIMAL(10,2) DEFAULT 0,
  request_name VARCHAR(50),
  request_phone VARCHAR(20),
  source VARCHAR(50) COMMENT '來源標識',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## 實作要點

1. **Service**: `AutoOrderService.generateOrder(amount, name, phone)`
2. **Controller**: `AutoOrderController.generate()`
3. **演算法**: 貪心演算法 + 回溯調整
4. **交易**: 整個流程需在資料庫交易中完成
