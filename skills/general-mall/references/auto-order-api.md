# 自動產生訂單 API

## 功能說明

提供 API 讓下游系統（站台）傳入金額，自動組合商品產生訂單。

- **代收（collect）**：產生購買訂單（status=3 已完成）
- **代付（pay）**：產生退貨申請（orderId=0，status=2 已完成）

## 認證方式：API Token

每個站台有獨立的 `api_token`，放在 HTTP Header：

```
X-Api-Token: <token>
```

驗證流程：
1. Token → 查 `oms_order_source`（啟用狀態）
2. employeeId → 驗證歸屬
3. deviceId → 驗證歸屬

## API 端點

| 功能 | 端點 |
|------|------|
| 代收 | `POST /order/auto-generate/collect` |
| 代付 | `POST /order/auto-generate/pay` |

## 請求參數

```json
{
  "employeeId": "emp001",
  "deviceId": "dev001",
  "amount": 300,
  "phone": "0900123456",
  "address": "林森北路厚德路",
  "remark": "",
  "notes": ""
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| employeeId | string | ✅ | 員工ID |
| deviceId | string | ✅ | 設備號 |
| amount | number | ✅ | 訂單金額（>0） |
| phone | string | ❌ | 手機號碼，未帶入自動產生（新增到假人庫）|
| address | string | ❌ | 收貨地址，未帶入自動帶 7-11 門市地址 |
| remark | string | ❌ | 備註 |
| notes | string | ❌ | 附註 |

### 可選欄位行為

- **phone**: 帶入→新增一筆到 `oms_fake_contact`（用隨機姓名+該電話+隨機門市）並使用；不帶→隨機取假人
- **address**: 帶入→覆蓋 `receiverDetailAddress`（縣市仍用門市的）；不帶→自動用「7-11 {門市名} ({地址})」
- **remark/notes**: 代收存到 `order.note`（用 `|` 分隔）；代付附加到退貨描述末尾

## 代收回應

```json
{
  "code": 200,
  "data": {
    "success": true,
    "type": "collect",
    "orderId": "202602131810477535",
    "items": [
      {"productId": 1106, "name": "商品A", "price": 223.0, "qty": 1}
    ],
    "subtotal": 311.0,
    "discount": 11.0,
    "total": 300
  }
}
```

## 代付回應

```json
{
  "code": 200,
  "data": {
    "success": true,
    "type": "pay",
    "returnOrderId": "R202602131810477535",
    "items": [...],
    "subtotal": 211.0,
    "adjustment": -11.0,
    "total": 200
  }
}
```

## 代付退貨架構

- **一筆 return_apply per 代付**（不再拆分多筆）
- `orderId = 0`（無關聯訂單）
- `orderSn = R + 流水號`
- 多商品明細存 `product_attr` 欄位（JSON 陣列）
- `returnAmount` = 目標金額
- `productCount` = 所有商品數量加總
- 退貨原因：從 `oms_order_return_reason` 隨機選取
- `source_name` 寫入站台名稱，供後台過濾

### product_attr JSON 格式

```json
[
  {
    "productId": 1106,
    "productName": "商品A",
    "productPrice": 223.00,
    "productCount": 1,
    "productPic": "http://...",
    "productBrand": "品牌A"
  }
]
```

### 前端判斷

- `orderId === 0` → 代付訂單
- 解析 `productAttr` JSON 顯示商品列表
- 隱藏物流/處理人區塊
- 問題描述用 `white-space: pre-wrap` 顯示換行

## 假人+門市系統

- `oms_fake_contact`: 10,000 筆台灣假人（2% 二字名，98% 三字名）
- `store_711`: 7,316 家 7-11 門市（22 縣市）
- 假人按人口比例綁定門市（`store_id` FK）
- `selectRandomWithStore()` JOIN 查詢一次取出假人+門市資訊
- 代收訂單自動帶入：receiverProvince=縣市, receiverCity=區域, receiverDetailAddress="7-11 {名} ({址})"

## 站台帳號系統

- `oms_order_source`: id, name(UNIQUE), api_token, status
- 建站台時指定 adminUsername/adminPassword → 自動建 `ums_admin` 帳號
- `ums_admin.source_id` 關聯站台
- 「站台角色」(ums_role): resource 7,8,30,31,32; menu 7,8,10
- 站台帳號只能看自己站台的訂單和退貨申請

## 商品組合算法

1. 取得所有上架商品（Redis 緩存 10 分鐘）
2. 多輪多樣性選擇（`selectProductsMultiRound`）
3. 每商品每輪 7~12 件限制
4. 折扣 ≤ 最低價商品
5. subtotal ≥ targetAmount，discount = subtotal - targetAmount

## 資料庫相關表

| 表 | 說明 |
|---|------|
| oms_order | 訂單主表 |
| oms_order_item | 訂單商品明細 |
| oms_order_source | 站台配置 |
| oms_order_source_device | 設備管理 |
| oms_order_source_employee | 員工管理 |
| oms_order_return_apply | 退貨申請（代付） |
| oms_order_return_reason | 退貨原因（8 種） |
| oms_fake_contact | 假人資料 |
| store_711 | 7-11 門市 |

## Postman Collection

`/home/ubuntu/general-mall/postman/Auto-Order-API.postman_collection.json` — 10 個請求

## 測試

`/home/ubuntu/general-mall/tests/run-tests.sh` — 78 tests

---
*Last updated: 2026-02-14*
