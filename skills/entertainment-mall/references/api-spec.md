# 娱乐城 API 对接规格

## 状态

⏳ **待对接** - API 文档待娱乐城方提供

## 需要对接的接口

### 1. 用户验证

**用途**: 用户登入时验证娱乐城帐号

```
POST /api/auth/verify
```

**请求**:
```json
{
  "username": "string",
  "password": "string",
  "casino_code": "string"
}
```

**响应**:
```json
{
  "success": true,
  "user_id": "string",
  "nickname": "string",
  "points": 10000.00
}
```

---

### 2. 查询点数

**用途**: 显示用户可用点数

```
GET /api/points/balance?user_id={user_id}
```

**响应**:
```json
{
  "success": true,
  "user_id": "string",
  "points": 10000.00
}
```

---

### 3. 扣除点数

**用途**: 下单时扣除用户点数

```
POST /api/points/deduct
```

**请求**:
```json
{
  "user_id": "string",
  "points": 1000.00,
  "order_id": "string",
  "description": "商品兑换"
}
```

**响应**:
```json
{
  "success": true,
  "transaction_id": "string",
  "remaining_points": 9000.00
}
```

---

### 4. 退还点数

**用途**: 订单取消/退货时退还点数

```
POST /api/points/refund
```

**请求**:
```json
{
  "user_id": "string",
  "points": 1000.00,
  "original_transaction_id": "string",
  "reason": "订单取消"
}
```

**响应**:
```json
{
  "success": true,
  "transaction_id": "string",
  "remaining_points": 10000.00
}
```

---

### 5. 发放彩金

**用途**: 虚拟商品（彩金）兑换后加到娱乐城余额

```
POST /api/bonus/grant
```

**请求**:
```json
{
  "user_id": "string",
  "amount": 1000.00,
  "order_id": "string",
  "product_name": "彩金1000"
}
```

**响应**:
```json
{
  "success": true,
  "transaction_id": "string",
  "new_balance": 5000.00
}
```

---

## 安全机制

### 认证方式
待确认（API Key / OAuth / 签名验证）

### 请求签名
```
待补充签名算法
```

### IP 白名单
商城服务器 IP 需加入娱乐城白名单

---

## 错误处理

```json
{
  "success": false,
  "error_code": "INSUFFICIENT_POINTS",
  "error_message": "点数不足"
}
```

### 常见错误码

| 错误码 | 说明 |
|--------|------|
| INVALID_USER | 用户不存在 |
| INVALID_TOKEN | 认证失败 |
| INSUFFICIENT_POINTS | 点数不足 |
| DUPLICATE_TRANSACTION | 重复交易 |
| SYSTEM_ERROR | 系统错误 |

---

## 待补充

- [ ] 娱乐城 API 文档
- [ ] 认证方式确认
- [ ] 签名算法
- [ ] 测试环境地址
- [ ] 正式环境地址
