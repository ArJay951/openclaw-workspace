# 部署指南

## 測試環境

| 項目 | 值                                |
|------|----------------------------------|
| 伺服器 IP | 52.76.231.27                     |
| 網域 | dev.homely-go.com                |
| 後台 | https://dev.homely-go.com/admin/ |
| 前台 | https://dev.homely-go.com/web/   |
| 登入帳號 | admin / homelygo5566             |

## 部署腳本

**位置**: `/home/ubuntu/general-mall/deploy.sh`

```bash
# 查看用法
bash deploy.sh help

# 部署選項
bash deploy.sh backend   # 更新後端（Java + Docker）
bash deploy.sh admin     # 更新後台前端（Vue）
bash deploy.sh web       # 更新前台前端（H5）
bash deploy.sh all       # 更新全部
bash deploy.sh db        # 重新匯入資料庫（⚠️ 會清空資料）
bash deploy.sh env       # 顯示當前環境配置
```

## 環境配置

**位置**: `/home/ubuntu/general-mall/env.conf`

```bash
# 後台前端 API 地址
ADMIN_API_URL="https://dev.homely-go.com/api/admin"

# 前台前端 API 地址
PORTAL_API_URL="https://dev.homely-go.com/api/portal"

# 商城名稱
MALL_NAME="商城系統"
```

修改後執行 `bash deploy.sh admin` 即可套用。

## Docker 服務管理

```bash
# 進入專案目錄
cd /home/ubuntu/general-mall

# 啟動所有服務
docker compose up -d

# 查看服務狀態
docker compose ps

# 查看日誌
docker compose logs -f mall-admin
docker compose logs -f mall-portal

# 重啟服務
docker compose restart mall-admin mall-portal
```

## 健康檢查

**腳本**: `/home/ubuntu/general-mall/health_check.sh`

```bash
bash /home/ubuntu/general-mall/health_check.sh
```

## 監控 Cron Job

- **ID**: `017ac75c-8511-402e-9c4c-880001a81af5`
- **頻率**: 每 5 分鐘
- **異常通知**: Telegram

## SSL 設定

使用 CDN 代理模式：
- CDN 處理 HTTPS (SSL 終止)
- 伺服器維持 HTTP (port 80)
- 無需在伺服器安裝憑證

## 資料庫連線

```bash
# MySQL
docker exec -it mall-mysql mysql -uroot -proot mall

# Redis
docker exec -it mall-redis redis-cli
```

## 手動重建後端

```bash
cd /home/ubuntu/general-mall/mall
mvn clean package -DskipTests
mvn docker:build -pl mall-admin,mall-portal
cd ..
docker compose up -d --force-recreate mall-admin mall-portal
```
