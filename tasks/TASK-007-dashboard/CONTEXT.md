# CONTEXT.md — TASK-007

## 環境
- **伺服器**: Ubuntu, 52.76.231.27
- **Node.js**: v22.22.0
- **PM2**: 已安裝（`pm2` 全域指令）
- **Nginx**: 已安裝運行中

## 路徑
- **專案目錄**: `/home/ubuntu/task-dashboard/`（需新建）
- **Workspace**: `/home/ubuntu/.openclaw/workspace/`
- **Tasks 目錄**: `/home/ubuntu/.openclaw/workspace/tasks/`
- **Archive**: `/home/ubuntu/.openclaw/workspace/tasks/archive/`
- **Memory**: `/home/ubuntu/.openclaw/workspace/memory/`

## OpenClaw Gateway
- **Port**: 18789
- **Hook endpoint**: `http://127.0.0.1:18789/hooks/agent`
- **Gateway token**: 需要從 config 讀取
  - Config 路徑: `~/.openclaw/config.json` 或 `~/.openclaw/config.json5`
  - 看 `gateway.auth.token` 欄位
  - 如果 hooks 未啟用，在 DONE.md 記錄需要的配置變更

## 技術選型
- **前端**: Vue 3 + Vite + vuedraggable-next (for kanban drag) + marked (markdown render)
- **後端**: Express + better-sqlite3 (sync SQLite driver) + jsonwebtoken + bcryptjs
- **SQLite DB 路徑**: `/home/ubuntu/task-dashboard/data/dashboard.db`
- **Port**: 3100

## 已安裝的全域套件
- node, npm, pm2
- 其他需要的自己 npm install

## ⚠️ 注意事項
- 不要動 mall 相關的任何東西
- 不要自己重啟 OpenClaw gateway
- SQLite 用 better-sqlite3（同步 API，簡單）
- 前端 build 後的 dist/ 由 Express static middleware 直接 serve
- JWT secret 用隨機生成，存在 .env 檔案
- 密碼用 bcryptjs hash
- Chat 如果 hooks 沒啟用，前端顯示「Chat 功能需要設定 OpenClaw hooks」的提示
