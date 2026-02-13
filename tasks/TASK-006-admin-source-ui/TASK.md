# TASK-006: 後台來源管理 UI 改造

## 目標
配合 TASK-004 的後端改動，更新後台前端的訂單來源管理頁面：
1. 移除 IP 白名單，改為顯示 API Token
2. 新增設備管理（員工ID + 設備號）
3. 更新 API 層

## ⚠️ 不做的事
- 不改後端（已完成）
- 不改其他頁面

---

## Step 1: 更新 API 層

### 修改 `src/api/orderSource.js`
新增以下 API：

```javascript
// 重新生成 Token
export function regenerateToken(id) {
  return request({ url: '/orderSource/regenerateToken/' + id, method: 'post' })
}

// 設備管理
export function fetchDevices(sourceId) {
  return request({ url: '/orderSource/' + sourceId + '/devices', method: 'get' })
}

export function createDevice(sourceId, data) {
  return request({ url: '/orderSource/' + sourceId + '/devices/create', method: 'post', data })
}

export function updateDevice(sourceId, id, data) {
  return request({ url: '/orderSource/' + sourceId + '/devices/update/' + id, method: 'post', data })
}

export function deleteDevice(sourceId, id) {
  return request({ url: '/orderSource/' + sourceId + '/devices/delete/' + id, method: 'post' })
}
```

---

## Step 2: 改造來源列表頁 `src/views/oms/source/index.vue`

### 表格欄位改為：
| 欄位 | 說明 |
|------|------|
| ID | 不變 |
| 來源名稱 | 不變 |
| API Token | 顯示遮罩（`****...xxxx` 只顯示後4位），旁邊有「複製」和「重新生成」按鈕 |
| 設備數量 | 顯示該來源底下有多少設備（可點擊展開或跳轉） |
| 狀態 | 不變（switch） |
| 建立時間 | 不變 |
| 操作 | 編輯、刪除、管理設備 |

### 新增/編輯對話框改為：
- 來源名稱（必填）
- 狀態
- **移除 IP 白名單輸入**
- 新增時不用填 Token（後端自動生成），建立成功後顯示完整 Token 一次

### Token 操作：
- 「複製」按鈕：用 `navigator.clipboard.writeText()` 複製完整 token
- 「重新生成」按鈕：確認對話框 → 呼叫 `regenerateToken` API → 顯示新 token

---

## Step 3: 設備管理

### 方案：在來源列表點「管理設備」→ 彈出設備管理對話框

設備對話框內容：
- 標題：「{來源名稱} - 設備管理」
- 表格：員工ID、設備號、狀態、建立時間、操作（編輯/刪除）
- 新增設備按鈕 → 小表單：員工ID（必填）、設備號（必填）、狀態
- 支援批量新增（可選，一個員工ID多個設備號用換行分隔）

---

## Step 4: Build + 部署

```bash
cd /home/ubuntu/mall-admin-web
npm run build:prod
```

確認 build 成功，`dist/` 目錄更新（nginx 直接讀 dist，不需重啟）。

---

## 驗收標準
1. `npm run build:prod` 成功
2. 來源列表顯示 Token（遮罩）、設備數量
3. 可複製 Token、可重新生成 Token
4. 可新增/編輯/刪除設備
5. 新增來源時自動生成 Token 並顯示
6. 表單已移除 IP 白名單
