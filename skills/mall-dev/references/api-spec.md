# 外部 API 接口规格

> 状态：预留设计，待外部系统确认后调整

## 1. ExternalAuthService（登录/注册）

### 配置项

```yaml
external:
  auth:
    enabled: true
    base-url: ${EXTERNAL_AUTH_URL:http://localhost:8080}
    login-path: /api/auth/login
    register-path: /api/auth/register
    validate-path: /api/auth/validate
    user-info-path: /api/auth/user
    timeout: 5000
```

### 接口定义

#### POST /api/auth/login
```json
// Request
{
  "username": "string",
  "password": "string"
}

// Response
{
  "code": 200,
  "data": {
    "token": "string",
    "externalUserId": "string",
    "expiresIn": 3600
  }
}
```

#### POST /api/auth/validate
```json
// Request Header
Authorization: Bearer <token>

// Response
{
  "code": 200,
  "data": {
    "valid": true,
    "externalUserId": "string",
    "nickname": "string",
    "avatar": "string"
  }
}
```

---

## 2. ExternalPointsService（点数余额/充值）

### 配置项

```yaml
external:
  points:
    enabled: true
    base-url: ${EXTERNAL_POINTS_URL:http://localhost:8080}
    balance-path: /api/points/balance
    recharge-path: /api/points/recharge
    history-path: /api/points/history
    callback-secret: ${POINTS_CALLBACK_SECRET}
```

### 接口定义

#### GET /api/points/balance?userId={externalUserId}
```json
// Response
{
  "code": 200,
  "data": {
    "externalUserId": "string",
    "balance": 100000,
    "updatedAt": "2026-02-03T06:00:00Z"
  }
}
```

#### POST /api/points/recharge
```json
// Request
{
  "externalUserId": "string",
  "amount": 10000,
  "callbackUrl": "string"
}

// Response
{
  "code": 200,
  "data": {
    "rechargeOrderId": "string",
    "paymentUrl": "string"
  }
}
```

---

## 3. ExternalExchangeService（商品兑换/扣点）

### 配置项

```yaml
external:
  exchange:
    enabled: true
    base-url: ${EXTERNAL_EXCHANGE_URL:http://localhost:8080}
    precheck-path: /api/exchange/precheck
    execute-path: /api/exchange/execute
    rollback-path: /api/exchange/rollback
    timeout: 10000
    retry-times: 3
```

### 接口定义

#### POST /api/exchange/precheck
```json
// Request
{
  "externalUserId": "string",
  "points": 10000
}

// Response
{
  "code": 200,
  "data": {
    "sufficient": true,
    "currentBalance": 50000
  }
}
```

#### POST /api/exchange/execute
```json
// Request
{
  "externalUserId": "string",
  "orderId": "string",
  "points": 10000,
  "description": "兑换商品: iPhone 17"
}

// Response
{
  "code": 200,
  "data": {
    "transactionId": "string",
    "newBalance": 40000,
    "deductedAt": "2026-02-03T06:00:00Z"
  }
}
```

#### POST /api/exchange/rollback
```json
// Request
{
  "transactionId": "string",
  "reason": "订单取消"
}

// Response
{
  "code": 200,
  "data": {
    "refunded": true,
    "refundedPoints": 10000,
    "newBalance": 50000
  }
}
```

---

## Mock 实现策略

开发阶段使用 Mock 实现：

```java
@Profile("mock")
@Service
public class MockExternalAuthService implements ExternalAuthService {
    // 固定返回测试用户
    // token 验证始终通过
    // 点数余额固定 100000
}
```

切换方式：
- `application-mock.yml` 启用 mock profile
- 生产环境配置真实 URL
