# 自動產生訂單 API

## 功能說明

提供 API 讓下游系統傳入金額、姓名、手機號，自動根據後台商品產生訂單。

- **代收**：創建購買訂單（status=0 待付款）
- **代付**：創建退貨訂單（status=5 無效訂單/待退款）

## API 端點

| 環境 | URL |
|------|-----|
| 外部訪問 | `POST http://dev.homely-go.com/portal/order/auto-generate` |
| 內部訪問 | `POST http://localhost:8085/order/auto-generate` |

## 請求參數

```json
{
  "type": "collect",
  "amount": 30000,
  "name": "張三",
  "phone": "0912345678"
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| type | string | ✅ | `collect`=代收(購買) / `pay`=代付(退貨) |
| amount | number | ✅ | 訂單金額（最終金額，固定不變） |
| name | string | ✅ | 收件人/退貨人姓名 |
| phone | string | ✅ | 手機號碼 |

---

## IP 白名單驗證

API 會驗證請求來源 IP，需在 `oms_order_source` 表中配置白名單。

```sql
-- 查看白名單
SELECT * FROM oms_order_source;

-- 新增白名單
INSERT INTO oms_order_source (name, ip_whitelist, status, create_time) 
VALUES ('來源名稱', '192.168.1.1,10.0.0.0/8', 1, NOW());
```

- 支援單一 IP: `192.168.1.1`
- 支援 CIDR: `10.0.0.0/8`, `172.16.0.0/12`
- 多個用逗號分隔

---

## 商品組合算法（向上湊）

### 核心邏輯

1. **目標**：商品總價 ≥ 傳入金額
2. **折扣** = 商品總價 - 傳入金額
3. **訂單金額** = 傳入金額（固定）

### 算法流程

1. 取得所有上架商品，按價格降序
2. 貪心選擇商品，使 subtotal ≥ targetAmount
3. 優先選擇浪費最少的組合
4. 計算折扣/調整金額

---

## 代收（Collect）

### 成功回應

```json
{
  "code": 200,
  "message": "操作成功",
  "data": {
    "success": true,
    "type": "collect",
    "orderId": "202602110634133628",
    "items": [
      {"productId": 3, "name": "商品A", "price": 850.00, "qty": 35},
      {"productId": 4, "name": "商品B", "price": 240.00, "qty": 1}
    ],
    "subtotal": 30010.00,
    "discount": 10.00,
    "total": 30000
  }
}
```

### 範例

| 傳入金額 | 商品總價 | 折扣 | 訂單金額 |
|---------|---------|-----|---------|
| 30,000 | 30,100 | 100 | **30,000** |
| 3,000 | 3,000 | 0 | **3,000** |
| 117 | 118 | 1 | **117** |

---

## 代付（Pay）

### 成功回應

```json
{
  "code": 200,
  "message": "操作成功",
  "data": {
    "success": true,
    "type": "pay",
    "returnOrderId": "R202602110634133628",
    "items": [
      {"productId": 3, "name": "商品A", "price": 850.00, "qty": 35},
      {"productId": 4, "name": "商品B", "price": 240.00, "qty": 1}
    ],
    "subtotal": 30010.00,
    "adjustment": -10.00,
    "total": 30000
  }
}
```

### 範例

| 傳入金額 | 商品總價 | 調整 | 退款金額 |
|---------|---------|------|---------|
| 30,000 | 30,100 | -100 | **30,000** |
| 3,000 | 3,000 | 0 | **3,000** |

---

## 錯誤回應

```json
{
  "code": 500,
  "message": "IP 不在白名單中: 1.2.3.4"
}
```

### 錯誤訊息

| 錯誤 | 說明 |
|------|------|
| IP 不在白名單中 | 請求 IP 未授權 |
| 金額不足，最低需要 X 元 | 金額低於最低商品價格 |
| 無上架商品 | 沒有可用商品 |
| 商品庫存不足 | 庫存無法湊到目標金額 |
| type 必須是 collect 或 pay | 類型參數錯誤 |

---

## 資料庫

### 訂單狀態

- **代收訂單**: `status = 0` (待付款)
- **代付訂單**: `status = 5` (無效訂單/待退款)

### 相關表

- `oms_order` - 訂單主表
- `oms_order_item` - 訂單商品明細
- `oms_order_source` - IP 白名單配置
- `oms_order_return_apply` - 退貨申請（代付會建立）

---

## Postman 測試集

檔案位置：`/home/ubuntu/mall-deploy/Auto-Order-API.postman_collection.json`

**變數設定：**
```
baseUrl = http://dev.homely-go.com/portal
```

**測試金額：** 3000, 3500, 3600, 5800, 7200, 9000, 10000, 10900, 11000, 11400, 12000, 15100, 15500, 17000, 18000, 19000, 20000, 22600, 27000, 28000, 30000, 31000, 40000, 42000, 50000, 100000

---

## 更新日誌

- **2026-02-11**: 修正算法為「向上湊」，訂單金額 = 傳入金額
- **2026-02-10**: 新增 IP 白名單驗證、source_domain 欄位
- **2026-02-06**: 初版實作
