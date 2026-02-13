# TASK-007: 任務面板 Dashboard（Kanban + Chat）

## 目標
建立一個 Web Dashboard，功能：
1. **Kanban 看板** — 待做 / 進行中 / 完成，可拖拉
2. **Chat** — 在面板上直接跟 AI 助手對話（透過 OpenClaw webhook）
3. **任務雙來源** — 自動掃描 workspace `tasks/` 目錄 + 手動 CRUD
4. **每日回顧** — 讀取 `memory/` 日誌，顯示今日完成項目
5. **登入保護** — 帳密登入

## 技術方案
- **前端**: Vue 3 + Vite + 拖拉庫（vuedraggable-next 或 @vueuse/integrations）
- **後端**: Node.js (Express)
- **DB**: SQLite（輕量，dashboard 專用，不污染 mall 的 MySQL）
- **Chat**: 透過 OpenClaw Gateway hooks `/hooks/agent` 端點
- **部署**: PM2 管理 + Nginx reverse proxy

---

## Step 1: 專案初始化

### 1a. 建立專案目錄
```
/home/ubuntu/task-dashboard/
├── server/           # Express 後端
│   ├── index.js
│   ├── routes/
│   │   ├── auth.js
│   │   ├── tasks.js
│   │   ├── chat.js
│   │   └── daily.js
│   ├── db.js         # SQLite 初始化
│   └── scanner.js    # tasks/ 目錄掃描器
├── client/           # Vue 3 前端
│   ├── src/
│   │   ├── views/
│   │   │   ├── Login.vue
│   │   │   ├── Board.vue
│   │   │   ├── Chat.vue
│   │   │   └── DailyReview.vue
│   │   ├── components/
│   │   │   ├── KanbanColumn.vue
│   │   │   ├── TaskCard.vue
│   │   │   └── ChatWindow.vue
│   │   ├── api/
│   │   ├── router/
│   │   ├── store/
│   │   ├── App.vue
│   │   └── main.js
│   └── vite.config.js
├── package.json
└── ecosystem.config.js  # PM2 配置
```

### 1b. DB Schema (SQLite)
```sql
-- 任務表
CREATE TABLE tasks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  description TEXT,
  status TEXT DEFAULT 'todo' CHECK(status IN ('todo','doing','done')),
  source TEXT DEFAULT 'manual' CHECK(source IN ('manual','workspace')),
  workspace_path TEXT,          -- workspace tasks/ 的路徑（自動掃描用）
  priority INTEGER DEFAULT 0,   -- 排序用
  position INTEGER DEFAULT 0,   -- kanban 欄內排序
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  completed_at DATETIME
);

-- Chat 歷史
CREATE TABLE chat_messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  role TEXT NOT NULL CHECK(role IN ('user','assistant')),
  content TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 使用者（簡單帳密）
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL
);
```

---

## Step 2: 後端 API

### Auth
- `POST /api/auth/login` — 登入，回傳 JWT
- `POST /api/auth/register` — 首次設定帳密（只允許第一個用戶）
- Middleware: 除了 login/register，其他 API 都要 JWT

### Tasks
- `GET /api/tasks` — 列表（可過濾 status）
- `POST /api/tasks` — 新增
- `PUT /api/tasks/:id` — 更新（含拖拉更新 status + position）
- `DELETE /api/tasks/:id` — 刪除
- `POST /api/tasks/sync` — 手動觸發掃描 workspace tasks/
- `PUT /api/tasks/reorder` — 批量更新排序

### Tasks 自動掃描邏輯
- 掃描 `/home/ubuntu/.openclaw/workspace/tasks/` 目錄
- 每個 `TASK-XXX-*/TASK.md` 解析為一個任務
- 有 `DONE.md` → status = done
- 在 `archive/` 裡 → status = done
- 否則 → status = doing
- 掃描時跟 DB 同步（新增/更新，不刪除手動任務）

### Chat
- `POST /api/chat/send` — 發送訊息給 OpenClaw
  - 透過 HTTP POST 到 `http://127.0.0.1:18789/hooks/agent`
  - Header: `Authorization: Bearer <gateway_token>`
  - Body: `{ "message": "...", "name": "Dashboard Chat" }`
  - 存入 chat_messages 表
  - 回傳 AI 的回覆
- `GET /api/chat/history` — 取得歷史（分頁）

### Daily Review
- `GET /api/daily/today` — 讀取 `memory/YYYY-MM-DD.md` 的今日內容
- `GET /api/daily/:date` — 讀取指定日期
- 解析 markdown，回傳結構化的完成項目列表

---

## Step 3: 前端

### 頁面

#### Login.vue
- 簡潔登入表單
- 首次訪問無帳號時顯示註冊表單

#### Board.vue（主頁面）
- 三欄 Kanban：待做 | 進行中 | 完成
- 卡片可拖拉（vuedraggable-next）
- 卡片顯示：標題、來源標籤（手動/workspace）、建立時間
- 點卡片展開詳情（description）
- 右上角：+ 新增任務、🔄 同步 workspace
- workspace 來源的卡片有特殊標記

#### Chat.vue
- 右側滑出面板或獨立頁面
- 對話式 UI，支援 markdown 渲染
- 發送按鈕 + Enter 發送

#### DailyReview.vue
- 日曆選擇日期
- 顯示該天的 memory 日誌（markdown 渲染）
- 今日完成的任務列表

### UI 風格
- 乾淨簡約，深色/淺色切換
- 不要太花俏，重功能
- 響應式（桌面為主，手機可用）

---

## Step 4: 部署

### 4a. Build 前端
```bash
cd /home/ubuntu/task-dashboard/client
npm run build
```

### 4b. PM2 啟動後端
```bash
cd /home/ubuntu/task-dashboard
pm2 start ecosystem.config.js
pm2 save
```

### 4c. Nginx 配置
- 先用 port 3100 直接訪問
- Nginx 配置準備好但先不啟用（等域名）
- 前端 static files 由 Express 直接 serve（`dist/`）

### 4d. 初始帳號
- 不預設帳號，首次訪問時引導註冊

---

## Step 5: OpenClaw Hook 設定

需要確認 OpenClaw gateway hooks 是否啟用。如果沒有：
1. 讀取當前 gateway config
2. 啟用 hooks 並設定 token
3. 重啟 gateway

⚠️ **不要自己重啟 gateway**，把需要的 config 變更寫在 DONE.md，由主管處理。

---

## 驗收標準
1. `npm run build` 前端成功
2. PM2 啟動後端成功
3. 瀏覽器訪問 `http://52.76.231.27:3100`：
   - 可註冊/登入
   - Kanban 三欄顯示正常
   - 可新增/拖拉/刪除任務
   - 同步 workspace 能掃到 archive 裡的任務
   - Chat 部分：如果 hooks 未啟用，顯示提示訊息即可
   - Daily Review 能讀取 memory/ 檔案
4. 所有 API 有 JWT 保護
