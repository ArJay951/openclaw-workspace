# TASK-007 完成報告

## ✅ 完成項目

### 專案結構
- `/home/ubuntu/task-dashboard/` — 完整全端專案
- 後端: Express + better-sqlite3 + JWT + bcryptjs
- 前端: Vue 3 + Vite + vuedraggable + marked
- DB: SQLite at `/home/ubuntu/task-dashboard/data/dashboard.db`

### 後端 API（全部完成）
- `POST /api/auth/register` — 首次註冊（只允許一個用戶）
- `POST /api/auth/login` — JWT 登入
- `GET /api/auth/status` — 檢查是否已有帳號
- `GET /api/tasks` — 任務列表
- `POST /api/tasks` — 新增任務
- `PUT /api/tasks/:id` — 更新任務
- `DELETE /api/tasks/:id` — 刪除任務
- `PUT /api/tasks/reorder` — 批量排序（拖拉用）
- `POST /api/tasks/sync` — 掃描 workspace tasks/
- `POST /api/chat/send` — 發送聊天（會嘗試打 OpenClaw hooks）
- `GET /api/chat/history` — 聊天歷史
- `GET /api/daily/today` — 今日 memory
- `GET /api/daily/:date` — 指定日期 memory

### 前端頁面（全部完成）
- **Login.vue** — 登入/首次註冊
- **Board.vue** — Kanban 三欄（待做/進行中/完成），可拖拉，新增/編輯/刪除，同步 workspace
- **Chat.vue** — 對話式 UI，markdown 渲染
- **DailyReview.vue** — 日期選擇器 + memory 日誌顯示

### 部署
- PM2 啟動成功，process name: `task-dashboard`
- Port: 3100，監聽 0.0.0.0
- 前端 build 成功，由 Express static serve
- `http://52.76.231.27:3100` 可訪問

## ⚠️ OpenClaw Hooks 配置
- 找不到 `~/.openclaw/config.json` 或 `config.json5`
- Chat 功能會嘗試打 `http://127.0.0.1:18789/hooks/agent`
- 如果 hooks 未啟用，Chat 會顯示錯誤提示
- **需要主管確認 hooks 配置並手動處理**

## 驗收確認
- [x] `npm run build` 前端成功
- [x] PM2 啟動成功（pm2 save 已執行）
- [x] `curl http://localhost:3100/` 回傳 HTML
- [x] `curl http://localhost:3100/api/auth/status` 回傳 `{"hasUsers":false}`
- [x] 所有 API 有 JWT 保護（除 auth 端點）
