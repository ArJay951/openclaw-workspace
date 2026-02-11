# 自動訂單創建規則

> 通用規則，適用於所有商城系統（General Mall / Entertainment Mall）

## 核心規則

### 1. 商品多樣性
- **必須選擇多種商品**，不能只選一個商品
- 目標至少 **3 種以上** 商品（金額允許的情況下）
- 透過打亂商品順序增加隨機性

### 2. 商品數量限制
- 每個商品每輪最多 **7~12 單位**（隨機）
- 採用**多輪選擇**算法
- 只有當所有商品都達到限制仍無法滿足目標金額時，才允許繼續增加

### 3. 折扣金額限制
- 折扣金額**不得超過最便宜商品的價格**
- 確保訂單合理性，避免出現過大折扣
- 公式：`折扣 = 商品總價 - 目標金額`，且 `折扣 <= 最低價商品`

---

## 算法流程

```
輸入: targetAmount（目標金額）

1. 取得所有上架商品（從緩存或數據庫）
2. 計算最低價商品價格 (minPrice)
3. 驗證 targetAmount >= minPrice

4. 多輪選擇:
   maxSubtotal = targetAmount + minPrice  // 折扣上限
   
   while (subtotal < targetAmount) {
     shuffle(products)  // 每輪打亂順序
     
     for (每個商品) {
       qtyThisRound = random(7, 12)
       
       // 檢查折扣限制
       if (subtotal + qty*price > maxSubtotal) {
         調整數量
       }
       
       添加到訂單
     }
   }

5. 創建訂單記錄

輸出: 訂單號、商品明細、折扣金額
```

---

## 訂單金額字段

| 字段 | 含義 | 設置值 |
|------|------|--------|
| total_amount | 商品合計 | 商品總價 (subtotal) |
| pay_amount | 訂單總金額 | 商品總價 (subtotal) |
| discount_amount | 折扣金額 | 商品總價 - 目標金額 |

**後台計算公式：**
```
應付款金額 = payAmount - discountAmount = 目標金額
```

---

## 訂單狀態

| 類型 | 狀態 | status |
|------|------|--------|
| 代收 | 已支付（待發貨） | 1 |
| 代付 | 無效訂單（待退款） | 5 |

---

## Redis 緩存

商品列表使用 Redis 緩存以提升效能。

| 項目 | 值 |
|------|-----|
| Key | `auto_order:products` |
| TTL | 600 秒（10 分鐘） |

### 緩存策略
- 首次請求：查詢數據庫 → 存入 Redis
- 後續請求：直接從 Redis 獲取
- 10 分鐘後自動過期
- 產品更新時可手動清除

### 清除緩存
```bash
# Redis 命令
docker exec mall-redis redis-cli DEL auto_order:products

# 或調用 API
autoOrderService.clearProductCache();
```

---

## 代碼常量

```java
// 每輪每商品最多數量
private static final int MIN_QTY_PER_ROUND = 7;
private static final int MAX_QTY_PER_ROUND = 12;

// 緩存過期時間（秒）
private static final long CACHE_EXPIRE_SECONDS = 600;

// Redis Key
private static final String REDIS_KEY_PRODUCTS = "auto_order:products";
```

---

## 錯誤處理

| 錯誤碼 | 說明 |
|--------|------|
| AMOUNT_TOO_LOW | 金額 < 最低價商品 |
| NO_PRODUCTS | 無上架商品 |
| INSUFFICIENT_PRODUCTS | 商品不足以湊到目標金額 |

---

## 測試驗證

新功能上線前，需驗證以下測試案例：

| 金額 | 預期結果 |
|------|----------|
| 1,000 | 3+ 種商品，折扣 ≤ 20 |
| 10,000 | 多種商品，數量 7~12 |
| 50,000 | 26 種商品，數量均勻分布 |
| 100,000 | 所有商品，多輪累積 |

---

## 更新日誌

| 日期 | 變更 |
|------|------|
| 2026-02-11 | 簡化緩存配置為 static 常量 |
| 2026-02-11 | 新增 Redis 緩存（TTL 10 分鐘） |
| 2026-02-11 | 新增折扣限制（≤ 最低價商品） |
| 2026-02-11 | 建立通用規則文檔 |
