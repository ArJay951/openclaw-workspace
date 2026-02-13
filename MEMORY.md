# MEMORY.md - 長期記憶

## 阿杰

- 時區: Asia/Taipei (UTC+8)
- Telegram Chat ID: 7268541653
- 工作：電商平台開發

## 設計偏好 ⚠️
- **不喜歡大面積強調色**，偏好低調柔和
- 強調色只用在小元素（價格、按鈕），大區塊用白/灰/中性色
- 參考 lhshop.online 佈局風格
- 不喜歡太紅太突兀

## 技術偏好
- Docker Compose 部署
- 環境變數配置 (${VAR:default} 模式)
- utf8mb4 字符集
- 新需求需確認後再開發
- SSL: 使用 CDN，不直接用 Let's Encrypt

## 報價標準
- 後端日薪: NT$8,000
- 前端日薪: NT$6,000
- 平均: NT$7,000/天

## 專案概覽

### General Mall（通用商城）
- 域名: dev.homely-go.com
- 伺服器: 52.76.231.27
- 技術: Docker Compose, Spring Boot, Vue.js, MySQL, Redis, RabbitMQ, ES
- Ports: admin 8080, portal 8085
- DB: `mall`
- 功能: Auto-Order API (代收代付, IP 白名單)
- GitHub: ArJay951/mall-backend, mall-admin-web, mall-app-web, mall-deploy
- 狀態: **穩定運營**

### Entertainment Mall（娛樂城商城）
- 域名: ent.homely-go.com
- Ports: admin 8090, portal 8095
- DB: `ent_mall` (77 tables)
- 前端: Vue 2 + Vant 2（純 H5，非 uni-app）
- 路徑: 後端 `/home/ubuntu/entertainment-mall/`, 前台 `/home/ubuntu/ent-mall-app-web/`, 後台 `/home/ubuntu/ent-mall-admin-web/`
- GitHub: ArJay951/ent-mall-backend, ent-mall-admin-web, ent-mall-app-web
- 狀態: **SaaS 多租戶已上線**，等待 Casino API 文件
- 已完成: 後台金額→點數, 前台 RWD + 淺色主題, SaaS Phase 1（多租戶核心）
- SaaS 架構: ent_master DB + 動態數據源 + domain 解析租戶
- 開通腳本: `/home/ubuntu/entertainment-mall/scripts/create-tenant.sh`

## 工作模式
- 2026-02-12 起改為「主管+員工」模式
- Main 不寫 code，只管理任務和指派 sub-agent
- 任務工單放 `tasks/` 資料夾
- Skills 放 `skills/` 給 sub 參考

## 品質保證
- API 自動測試: general-mall 26 tests, entertainment-mall 23 tests
- 前後端相依表: `DEPENDENCY-MAP.md`
- 開工單前必查相依表

## Dashboard 任務面板
- 路徑: `/home/ubuntu/task-dashboard/`
- 訪問: `http://52.76.231.27:3100`（等域名）
- OpenClaw hooks token: `bf6a17123dd96c7ca8f0330cd5c7978f25bfdcd66b894d48`

## Skills 位置
- general-mall: ~/.openclaw/workspace/skills/general-mall/
- entertainment-mall: ~/.openclaw/workspace/skills/entertainment-mall/

---
*Last updated: 2026-02-12*
