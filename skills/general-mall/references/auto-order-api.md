# 自動產生訂單 API

## 功能說明

提供 API 讓下游系統傳入金額、姓名、手機號，自動根據後台商品產生訂單。

- **代收**：創建購買訂單
- **代付**：創建退貨訂單

## API 規格

### 請求

```
POST /api/order/auto-generate
```

```json
{
  "type": "collect",
  "amount": 117,
  "name": "張三",
  "phone": "0912345678"
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| type | string | ✅ | `collect`=代收(購買) / `pay`=代付(退貨) |
| amount | number | ✅ | 訂單金額 |
| name | string | ✅ | 收件人/退貨人姓名 |
| phone | string | ✅ | 手機號碼 |

---

## 代收（購買訂單）

### 處理邏輯

1. 從所有商品中選擇，按價格降序
2. 優先選高價商品，同一商品可多件
3. 剩餘金額若低於最低價商品 → 產生折扣券
4. 折扣不超過最低價商品金額
5. 金額不足最低價商品 → 返回錯誤

### 成功回應

```json
{
  "success": true,
  "type": "collect",
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

### 示例

| 商品價格 | 20元、50元、100元 |
|----------|-------------------|
| 傳入金額 | 117元 |
| 產生訂單 | 100元×1 + 20元×1 |
| 小計 | 120元 |
| 折扣 | 3元 |
| 實付 | 117元 |

---

## 代付（退貨訂單）

### 處理邏輯

1. 從所有商品中選擇，按價格降序
2. 組合出符合金額的商品組合
3. 剩餘金額若低於最低價商品 → 產生補貼
4. 補貼不超過最低價商品金額
5. 創建退貨訂單，狀態為「待退款」

### 成功回應

```json
{
  "success": true,
  "type": "pay",
  "return_order_id": "R202602060001",
  "items": [
    {"product_id": 3, "name": "100元商品", "price": 100, "qty": 1},
    {"product_id": 1, "name": "20元商品", "price": 20, "qty": 1}
  ],
  "subtotal": 120,
  "adjustment": -3,
  "total": 117
}
```

### 示例

| 商品價格 | 20元、50元、100元 |
|----------|-------------------|
| 傳入金額 | 117元 |
| 產生退貨 | 100元×1 + 20元×1 |
| 小計 | 120元 |
| 調整 | -3元 |
| 退款金額 | 117元 |

---

## 錯誤回應

```json
{
  "success": false,
  "error_code": "AMOUNT_TOO_LOW",
  "error_message": "金額不足，最低需要 20 元"
}
```

### 錯誤碼

| 錯誤碼 | 說明 |
|--------|------|
| INVALID_TYPE | type 必須是 collect 或 pay |
| AMOUNT_TOO_LOW | 金額不足最低價商品 |
| NO_PRODUCTS | 無上架商品 |
| INVALID_PARAMS | 參數缺失或格式錯誤 |

---

## 資料庫擴展

```sql
-- 自動訂單記錄表
CREATE TABLE oms_auto_order_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_type VARCHAR(20) NOT NULL COMMENT 'collect=代收 pay=代付',
  order_id BIGINT COMMENT '購買訂單ID',
  return_order_id BIGINT COMMENT '退貨訂單ID',
  request_amount DECIMAL(10,2) NOT NULL,
  actual_amount DECIMAL(10,2) NOT NULL,
  adjustment_amount DECIMAL(10,2) DEFAULT 0 COMMENT '折扣或調整金額',
  request_name VARCHAR(50),
  request_phone VARCHAR(20),
  source VARCHAR(50) COMMENT '來源標識',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 實作要點

### Service

```java
public interface AutoOrderService {
    // 代收 - 創建購買訂單
    CollectOrderResult generateCollectOrder(BigDecimal amount, String name, String phone);
    
    // 代付 - 創建退貨訂單  
    PayOrderResult generatePayOrder(BigDecimal amount, String name, String phone);
}
```

### 商品組合演算法

1. 取得所有上架商品，按價格降序排列
2. 貪心演算法填滿金額
3. 計算剩餘金額，決定折扣/調整
4. 驗證折扣不超過最低價商品

### 交易處理

- 整個流程需在資料庫交易中完成
- 代收：創建訂單 → 鎖定庫存
- 代付：創建退貨單 → 記錄待退款
