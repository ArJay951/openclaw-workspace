---
name: entertainment-mall
description: 娛樂城積分兌換商城系統開發指南。基於 macrozheng/mall 框架，對接娛樂城平台 API，支援多套獨立部署。用於：(1) 商城系統開發與維護 (2) 功能需求討論 (3) 部署與配置 (4) 娛樂城 API 對接
---

# 娛樂城商城系統 (Entertainment Mall)

## 專案概述

獨立的積分兌換電商平台，提供 API 對接娛樂城平台。支援多套獨立部署，每套服務一個娛樂城。

## 🔤 語言規範

**本專案使用繁體中文**

- 程式碼註解、訊息、UI 文字一律使用繁體中文
- 若阿杰輸入簡體中文，執行時自動轉換為繁體

## ⚠️ 開發流程

**新增需求時，必須先確認再開發：**

1. **記錄需求** → 寫入 `references/requirements.md`
2. **確認需求** → 與阿杰確認理解是否正確
3. **開發實作** → 確認後才開始寫程式
4. **測試驗證** → 完成後回報結果

❌ 禁止：收到需求直接開發
✅ 正確：先確認、再開發

## 📋 新增頁面檢查清單

新增後台頁面時，必須完成以下步驟：

1. **後端**
   - [ ] Controller + Service + Mapper
   - [ ] 資料庫表（如需要）

2. **前端**
   - [ ] API 檔案 (`src/api/xxx.js`)
   - [ ] 頁面元件 (`src/views/xxx/index.vue`)
   - [ ] 路由配置 (`src/router/index.js`)
   - [ ] 重新構建 (`npm run build`)

3. **權限設定**（⚠️ 必須）
   - [ ] `ums_menu` - 新增菜單
   - [ ] `ums_role_menu_relation` - 給超級管理員(id=5)添加菜單權限
   - [ ] `ums_resource` - 新增 API 資源
   - [ ] `ums_role_resource_relation` - 給超級管理員添加資源權限
   - [ ] 清除 Redis 快取 (`docker exec mall-redis redis-cli FLUSHALL`)
   - [ ] 重啟 mall-admin

## 技術架構

### 後端
- **框架**: Spring Boot + MyBatis
- **資料庫**: MySQL
- **快取**: Redis
- **搜尋**: Elasticsearch
- **訊息佇列**: RabbitMQ
- **部署**: Docker 容器化

### 前端
- **後台管理**: Vue + Element UI
- **前台商城**: uni-app（樣式參考 lhshop.online）

## 核心模組

| 模組 | 來源 | 說明 |
|------|------|------|
| mall-admin | mall | 後台管理 API |
| mall-portal | mall | 前台商城 API |
| mall-common | mall | 通用工具 |
| mall-security | mall (改造) | 安全認證，對接娛樂城 |
| mall-mbg | mall | MyBatis 程式碼生成 |
| mall-search | mall | Elasticsearch 搜尋 |

## 需要改造的部分

### 1. 用戶認證
- 原: 手機號/郵箱註冊登入
- 改: 對接娛樂城 API 認證

### 2. 支付系統
- 原: 支付寶/微信支付
- 改: 點數扣除（呼叫娛樂城 API）

### 3. 虛擬商品
- 新增: 彩金兌換 → 呼叫娛樂城 API 發放餘額

### 4. 多租戶配置
- 新增: 商城名稱、Logo、API 端點可配置

## 部署規範

### 環境分離
- **測試環境**: 可自由更新部署
- **正式環境**: 🔒 需授權才能更新

### Docker 部署
每套商城獨立容器，包含：
- 應用服務 (mall-admin, mall-portal)
- MySQL
- Redis
- Elasticsearch
- RabbitMQ
- Nginx

## 參考文件

- [需求邊界](references/requirements.md) - 完整需求規格
- [技術架構](references/architecture.md) - 詳細架構設計
- [資料庫設計](references/database.md) - 資料庫表結構
- [API 規格](references/api-spec.md) - 娛樂城對接 API

### 通用規則（共用 general-mall）
- [📋 訂單創建規則](../general-mall/references/order-rules.md) - 多樣性、數量限制、折扣限制、緩存

## 開發狀態

🟡 **等待娛樂城 API 文檔**

娛樂城商城開發需要對方提供 API 文檔後才能進行。

## 源碼倉庫

### 娛樂城商城 Repos
| 倉庫 | 說明 | GitHub |
|------|------|--------|
| ent-mall-backend | 後端 Java | https://github.com/ArJay951/ent-mall-backend |
| ent-mall-admin-web | 後台前端 | https://github.com/ArJay951/ent-mall-admin-web |

### 本機路徑
| 項目 | 路徑 |
|------|------|
| 後端 | `/home/ubuntu/entertainment-mall/` |
| 後台前端 | `/home/ubuntu/ent-mall-admin-web/` |
| 數據庫腳本 | `/home/ubuntu/entertainment-mall/document/sql/mall.sql` |

### 部署隔離（與 general-mall 分離）
| 資源 | 娛樂城商城 | 支付商城 |
|------|-----------|---------|
| Admin Port | 8090 | 8080 |
| Portal Port | 8095 | 8085 |
| MySQL DB | `ent_mall` | `mall` |
| Redis Prefix | `ent:` | `mall:` |
| Docker 前綴 | `ent-*` | `mall-*` |
