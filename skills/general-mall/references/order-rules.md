# 自動訂單創建規則

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
1. 取得所有上架商品
2. 計算最低價商品價格 (minPrice)
3. 驗證目標金額 >= minPrice

4. 多輪選擇:
   while (商品總價 < 目標金額) {
     for (每個商品) {
       本輪數量 = random(7, 12)
       添加到訂單
     }
   }

5. 驗證折扣:
   折扣 = 商品總價 - 目標金額
   if (折扣 > minPrice) {
     調整商品組合，減少折扣
   }

6. 創建訂單
```

---

## 訂單金額字段

| 字段 | 含義 | 設置值 |
|------|------|--------|
| total_amount | 商品合計 | 商品總價 (subtotal) |
| pay_amount | 訂單總金額 | 商品總價 (subtotal) |
| discount_amount | 折扣金額 | 商品總價 - 目標金額 |
| **應付款金額** | 後台顯示 | pay_amount - discount_amount |

**後台計算公式：**
```
應付款金額 = payAmount - discountAmount = 目標金額
```

---

## 範例

### 目標金額 1000 元

| 商品 | 單價 | 數量 | 小計 |
|------|------|------|------|
| 商品 A | 65 | 8 | 520 |
| 商品 B | 65 | 8 | 520 |
| **合計** | | | **1040** |
| **折扣** | | | **40** |
| **應付** | | | **1000** |

✅ 符合規則：
- 2 種商品（多樣性）
- 每種 8 件（7~12 範圍內）
- 折扣 40 元 < 最低價 65 元

---

## 錯誤處理

| 情況 | 處理 |
|------|------|
| 金額 < 最低價商品 | 返回錯誤 `AMOUNT_TOO_LOW` |
| 無上架商品 | 返回錯誤 `NO_PRODUCTS` |
| 商品不足以湊到目標 | 返回錯誤 `INSUFFICIENT_PRODUCTS` |
| 折扣超過限制 | 調整商品組合 |

---

## 代碼位置

```
mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java
```

### 關鍵常量
```java
// 每輪每商品最多數量
private static final int MIN_QTY_PER_ROUND = 7;
private static final int MAX_QTY_PER_ROUND = 12;
```

---

## Redis 緩存

商品列表使用 Redis 緩存以提升效能。

| 項目 | 值 |
|------|-----|
| Key | `auto_order:products` |
| TTL | 600 秒（10 分鐘） |
| 配置項 | `auto-order.cache.expire` |

### 清除緩存

產品更新時可調用：
```java
autoOrderService.clearProductCache();
```

或直接用 Redis 命令：
```bash
docker exec mall-redis redis-cli DEL auto_order:products
```

---

## 更新日誌

- **2026-02-11**: 新增 Redis 緩存（TTL 10 分鐘）
- **2026-02-11**: 建立規則文檔
