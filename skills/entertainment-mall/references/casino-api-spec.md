# 娛樂城 API 對接規格

## 概述

本文檔定義商城與娛樂城平台對接的 API 規格。由我方設計，娛樂城方實作。

## 認證機制

```
Header:
  X-API-Key: {api_key}
  X-Timestamp: {unix_timestamp}
  X-Signature: {signature}

Signature = HMAC-SHA256(api_key + timestamp + request_body, api_secret)
```

## Base URL

```
測試環境: https://{casino-domain}/api/mall
正式環境: https://{casino-domain}/api/mall
```

---

## API 列表

### 1. 用戶驗證

驗證娛樂城帳號密碼，返回 token。

```
POST /user/verify

Request:
{
  "username": "player001",
  "password": "hashed_password"
}

Response (成功):
{
  "code": 200,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 7200,
    "user_id": "u_123456",
    "username": "player001",
    "nickname": "玩家一號",
    "points": 50000
  }
}

Response (失敗):
{
  "code": 401,
  "message": "帳號或密碼錯誤"
}
```

---

### 2. 用戶資訊

取得用戶資訊與點數餘額（需 token）。

```
GET /user/info

Header:
  Authorization: Bearer {token}

Response:
{
  "code": 200,
  "data": {
    "user_id": "u_123456",
    "username": "player001",
    "nickname": "玩家一號",
    "points": 50000,
    "level": "VIP1",
    "avatar": "https://..."
  }
}
```

---

### 3. 點數扣除

下單時扣除點數。

```
POST /points/deduct

Header:
  Authorization: Bearer {token}

Request:
{
  "order_no": "ORD202402110001",
  "amount": 1500,
  "description": "購買商品: iPhone 手機殼 x1"
}

Response (成功):
{
  "code": 200,
  "data": {
    "transaction_id": "TXN_789012",
    "order_no": "ORD202402110001",
    "amount": 1500,
    "balance_before": 50000,
    "balance_after": 48500
  }
}

Response (餘額不足):
{
  "code": 400,
  "message": "點數餘額不足",
  "data": {
    "required": 1500,
    "available": 1000
  }
}
```

---

### 4. 點數返還

訂單取消或退貨時返還點數。

```
POST /points/refund

Header:
  Authorization: Bearer {token}

Request:
{
  "order_no": "ORD202402110001",
  "transaction_id": "TXN_789012",
  "amount": 1500,
  "reason": "用戶取消訂單"
}

Response:
{
  "code": 200,
  "data": {
    "refund_id": "RFD_345678",
    "order_no": "ORD202402110001",
    "amount": 1500,
    "balance_after": 50000
  }
}
```

---

### 5. 彩金發放

虛擬商品（彩金）出貨時發放餘額。

```
POST /bonus/grant

Header:
  Authorization: Bearer {token}

Request:
{
  "order_no": "ORD202402110002",
  "order_item_id": 12345,
  "bonus_type": "FREE_CREDIT",
  "amount": 1000,
  "description": "兌換彩金 1000"
}

Response:
{
  "code": 200,
  "data": {
    "grant_id": "GRT_567890",
    "order_no": "ORD202402110002",
    "amount": 1000,
    "bonus_balance": 1000
  }
}
```

---

### 6. 訂單狀態通知（Callback）

商城主動通知娛樂城訂單狀態變更。

```
POST {casino_callback_url}/order/notify

Request:
{
  "order_no": "ORD202402110001",
  "status": "SHIPPED",
  "status_name": "已發貨",
  "tracking_no": "SF1234567890",
  "tracking_company": "順豐速運",
  "update_time": "2024-02-11T15:30:00+08:00"
}

狀態值:
- PENDING: 待付款
- PAID: 已付款
- SHIPPED: 已發貨
- COMPLETED: 已完成
- CANCELLED: 已取消
- REFUNDED: 已退款

Response:
{
  "code": 200,
  "message": "OK"
}
```

---

## 錯誤碼

| Code | 說明 |
|------|------|
| 200 | 成功 |
| 400 | 請求參數錯誤 |
| 401 | 認證失敗 |
| 403 | 權限不足 |
| 404 | 資源不存在 |
| 500 | 伺服器錯誤 |

---

## 注意事項

1. 所有金額單位為「點數」，整數
2. 時間格式為 ISO 8601（含時區）
3. 請求需在 30 秒內完成簽名驗證
4. 建議實作冪等性，避免重複扣款
