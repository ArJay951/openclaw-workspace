# CONTEXT.md — TASK-006

## 專案路徑
- **前端根目錄**: `/home/ubuntu/mall-admin-web/`
- **框架**: Vue 2 + Element UI
- **Build**: `npm run build:prod`
- **產出**: `dist/` 目錄（nginx alias 直接讀）

## 關鍵檔案
- `src/views/oms/source/index.vue` — 來源管理頁（要改造）
- `src/api/orderSource.js` — API 層（要新增）
- `src/router/index.js` — 路由（確認來源頁路由存在）
- `src/utils/request.js` — axios 封裝

## 後端 API 端點（已完成，可直接用）

### 來源管理
- `GET /orderSource/list` — 列表（回傳含 apiToken）
- `GET /orderSource/{id}` — 單筆
- `POST /orderSource/create` — 新增（body: {name, status}，回傳含自動生成的 apiToken）
- `POST /orderSource/update/{id}` — 更新
- `POST /orderSource/delete/{id}` — 刪除
- `POST /orderSource/updateStatus` — 更新狀態（params: id, status）
- `POST /orderSource/regenerateToken/{id}` — 重新生成 Token（回傳新 token string）

### 設備管理
- `GET /orderSource/{sourceId}/devices` — 列出設備
- `POST /orderSource/{sourceId}/devices/create` — 新增（body: {employeeId, deviceId, status}）
- `POST /orderSource/{sourceId}/devices/update/{id}` — 更新
- `POST /orderSource/{sourceId}/devices/delete/{id}` — 刪除

## OmsOrderSource Model 欄位
- id, name, apiToken, status, createTime, updateTime
- ⚠️ 已無 ipWhitelist

## OmsOrderSourceDevice Model 欄位
- id, sourceId, employeeId, deviceId, status, createTime, updateTime

## UI 框架
- Element UI（el-table, el-dialog, el-form, el-button, el-tag, el-switch 等）
- 複製到剪貼板用 `navigator.clipboard.writeText()` 或裝 `clipboard` 套件
