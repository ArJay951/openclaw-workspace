# CONTEXT.md — TASK-008

## API 端點一覽

### Portal (port 8085) — 代收代付
- `POST /order/auto-generate/collect` — 代收
- `POST /order/auto-generate/pay` — 代付
- `POST /order/auto-generate/refresh-cache` — 刷新緩存
- Header: `X-Api-Token: <token>`
- Body: `{ "employeeId": "...", "deviceId": "...", "amount": 100 }`

### Admin (port 8080) — 後台
登入:
- `POST /admin/login` — body: `{ "username": "admin", "password": "123456" }`
- 回傳: `{ "data": { "tokenHead": "Bearer ", "token": "xxx" } }`
- 後續請求 Header: `Authorization: Bearer <token>`

來源管理:
- `GET /orderSource/list`
- `GET /orderSource/{id}`
- `POST /orderSource/create` — body: `{ "name": "...", "status": 1 }`
- `POST /orderSource/update/{id}`
- `POST /orderSource/delete/{id}`
- `POST /orderSource/updateStatus` — params: id, status
- `POST /orderSource/regenerateToken/{id}`

設備管理:
- `GET /orderSource/{sourceId}/devices`
- `POST /orderSource/{sourceId}/devices/create` — body: `{ "employeeId": "...", "deviceId": "...", "status": 1 }`
- `POST /orderSource/{sourceId}/devices/update/{id}`
- `POST /orderSource/{sourceId}/devices/delete/{id}`

訂單管理:
- 訂單列表/詳情（查看現有的 OmsOrderController）
- `POST /order/update/receiverInfo` — body: `{ "orderId": ..., "receiverName": "...", "receiverPhone": "..." }`
- `POST /returnApply/update/returnInfo` — body: `{ "id": ..., "returnName": "...", "returnPhone": "..." }`

## 測試資料
- Admin: username=admin, password=123456
- API Token (claw): 36ae430b6120992e5ac779cd8713342e
- 測試員工: employeeId=emp001, deviceId=dev001, sourceId=2

## 路徑
- 測試腳本放: `/home/ubuntu/general-mall/tests/`
- 後端: `/home/ubuntu/general-mall/mall/`
