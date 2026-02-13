# TASK-006 完成報告

## 狀態：✅ 完成

## 改動摘要

### 1. API 層 (`src/api/orderSource.js`)
- 新增 `regenerateToken(id)` — 重新生成 Token
- 新增 `fetchDevices(sourceId)` — 列出設備
- 新增 `createDevice(sourceId, data)` — 新增設備
- 新增 `updateDevice(sourceId, id, data)` — 更新設備
- 新增 `deleteDevice(sourceId, id)` — 刪除設備

### 2. 來源列表頁 (`src/views/oms/source/index.vue`)
- **移除** IP 白名單欄位和表單輸入
- **新增** API Token 欄位（遮罩顯示 `****xxxx`）+ 複製/重新生成按鈕
- **新增** 設備數量欄位（可點擊）
- **新增** 操作欄「管理設備」按鈕
- 新增時不填 Token（後端自動生成），建立成功後彈窗顯示完整 Token
- 重新生成 Token 有確認對話框，成功後彈窗顯示新 Token

### 3. 設備管理對話框
- 「{來源名稱} - 設備管理」彈窗
- 表格：員工ID、設備號、狀態、建立時間、操作
- 新增設備支援批量（多個設備號換行分隔）
- 支援編輯、刪除

## 驗收
- ✅ `npm run build` 成功（`dist/` 已更新）
- ✅ 已 git commit

## 備註
- Build script 是 `npm run build`（非 `build:prod`），TASK.md 中的指令有誤但不影響結果
