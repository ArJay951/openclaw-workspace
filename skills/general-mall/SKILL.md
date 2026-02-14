---
name: general-mall
description: 通用商城系統開發指南。基於 macrozheng/mall 框架，保留完整功能，支援多套獨立部署。用於：(1) 商城系統部署與維護 (2) 多租戶配置 (3) AWS 環境部署
---

# 商城系統 (General Mall)

## 專案概述

一般電商平台，基於 mall 完整功能，支援多套獨立部署。

## 🔤 語言規範

**本專案使用繁體中文**

- 程式碼註解、訊息、UI 文字一律使用繁體中文
- 若阿杰輸入簡體中文，執行時自動轉換為繁體
- 資料庫初始資料已轉換為繁體 (`/home/ubuntu/general-mall/document/sql/mall.sql`)

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
   - [ ] 清除 Redis 緩存 (`docker exec mall-redis redis-cli FLUSHALL`)
   - [ ] 重啟 mall-admin

## 技術架構

### 後端
- **框架**: Spring Boot + MyBatis
- **資料庫**: MySQL (AWS RDS)
- **快取**: Redis
- **搜尋**: Elasticsearch
- **訊息佇列**: RabbitMQ
- **部署**: Docker + AWS

### 前端
- **後台管理**: Vue + Element UI
- **前台商城**: uni-app（原版）

## 功能範圍

### 保留 mall 完整功能
- 商品管理
- 訂單管理
- 會員管理
- 促銷管理
- 內容管理
- 權限管理
- Elasticsearch 搜尋

### 新增功能：自動產生訂單 API ✅

提供 API 讓站台傳入金額，自動組合商品產生訂單。

- **代收**: `POST /order/auto-generate/collect` → 購買訂單（status=3 已完成）
- **代付**: `POST /order/auto-generate/pay` → 退貨申請（orderId=0，一筆 return_apply）
- **認證**: API Token（X-Api-Token header）+ 員工/設備驗證
- **假人+門市**: 自動帶入台灣假人姓名電話 + 7-11 門市地址
- **可選參數**: phone, address, remark, notes

詳見：[自動產生訂單 API](references/auto-order-api.md)

### 站台帳號系統 ✅

- 每個站台有獨立 admin 帳號（用戶指定 username/password）
- 站台角色：只能看訂單+退貨申請，且只看自己站台的資料
- `oms_order_source.name` UNIQUE 約束

### 導出 Excel ✅

- 訂單列表: `/order/exportExcel`
- 退貨申請: `/returnApply/exportExcel`
- 驗證：至少一個篩選條件 + 日期區間 ≤ 31 天

## 多租戶配置

每套商城可獨立配置：
- 商城名稱
- 商城 Logo
- 資料庫連接
- 其他品牌設定

## 部署架構

- **測試機**: 單機 Docker 全包
- **正式機**: EC2 + RDS (Multi-AZ) + ALB

## ⚠️ 踩坑記錄（Sub-agent 必讀）

### MyBatis Mapper 規範
- **本專案統一使用 XML Mapper**（不用 `@Select`/`@Insert` 注解）
- XML 放在 `mall-mbg/src/main/resources/com/macro/mall/mapper/`
- 每個 Mapper 都要有 `resultMap` 明確定義欄位映射
- JOIN 查詢用 `extends="BaseResultMap"` 擴展（參考 `OmsFakeContactMapper.xml` 的 `WithStoreResultMap`）
- **禁止用 `@Select("SELECT * ...")` + 依賴駝峰映射**（本專案沒開 `mapUnderscoreToCamelCase`）
- 2026-02-14：已將 OmsOrderSource/OmsOrderSourceDevice/OmsFakeContact 三個 Mapper 從注解改為 XML

### YAML 配置
- 刪除 YAML 段落後，**檢查相鄰配置的縮排是否壞掉**
- 改完後用 `python3 -c "import yaml; yaml.safe_load(open('xxx.yml'))"` 驗證
- 關鍵路徑：`spring.redis.host`, `spring.rabbitmq.host`, `spring.datasource.url`
- 2026-02-13 事件：移除 MongoDB 段落後 Redis 縮排壞掉 → portal 503

### Docker 部署
- Jar 是 COPY 進 image 的（非 volume mount），改 code 要重建 image
- 每次部署後必做：health check → API 冒煙測試 → 日誌檢查
- Dockerfile 模板：`/tmp/Dockerfile-portal`, `/tmp/Dockerfile-admin`

### Redis 緩存
- 改角色權限後必須清 Redis：`DEL mall:ums:admin:{username}` + `mall:ums:resourceList:{roleId}`
- 或直接 `FLUSHALL`（dev 環境）

### poi-ooxml 版本
- 使用 **4.1.2**，5.2.5 與 commons-compress 衝突

### Lombok
- Model/DTO 類用 Lombok（`@Data` 等），不手寫 getter/setter

## 導出 Excel 模式

兩個導出 API（訂單 + 退貨申請）用相同模式：

```java
// 1. 驗證：至少一個篩選條件 + 日期區間 ≤ 31 天
// 2. 查詢：listAll() 不用 PageHelper
// 3. 寫 Excel：hutool ExcelUtil.getWriter(true) + addHeaderAlias + write
// 4. wrapText：商品明細欄 CellStyle.setWrapText(true)
// 5. 回傳：Content-Type xlsx + Content-Disposition attachment
```

前端：
```js
// axios 設 responseType: 'blob'
// request.js interceptor 判斷 response.config.responseType === 'blob' 回傳 raw response
// URL.createObjectURL + <a> download
```

## 參考文件

- [需求邊界](references/requirements.md)
- [環境規格](references/environment.md)
- [自動產生訂單 API](references/auto-order-api.md)
- [待開發功能](references/pending-features.md)
- [部署指南](references/deployment.md) ⭐ 新增

## 源碼倉庫

| 倉庫 | 說明 | GitHub |
|------|------|--------|
| mall-backend | 後端 Java | https://github.com/ArJay951/mall-backend |
| mall-admin-web | 後台前端 | https://github.com/ArJay951/mall-admin-web |
| mall-app-web | 前台 H5 | https://github.com/ArJay951/mall-app-web |
| mall-deploy | 部署配置 | https://github.com/ArJay951/mall-deploy |

### 本機路徑
- 專案目錄: `/home/ubuntu/general-mall/`
- 後端源碼: `/home/ubuntu/general-mall/mall/`
- 後台前端: `/home/ubuntu/mall-admin-web/`
- 前台前端: `/home/ubuntu/mall-app-build/`

## 測試環境

| 項目 | 值 |
|------|-----|
| 伺服器 IP | 52.76.231.27 |
| 網域 | dev.homely-go.com |
| 後台 | https://dev.homely-go.com/admin/ |
| 前台 | https://dev.homely-go.com/web/ |
| 登入帳號 | admin / macro123 |
| SSL | CDN 代理模式 |

### 服務端口
| 服務 | 端口 |
|------|------|
| MySQL | 3306 |
| Redis | 6379 |
| RabbitMQ | 5672/15672 |
| Elasticsearch | 9200 |
| mall-admin | 8080 |
| mall-portal | 8085 |
| Nginx | 80 |

### 快速指令
```bash
# 部署腳本
bash /home/ubuntu/general-mall/deploy.sh [backend|admin|web|all|db|env]

# 健康檢查
bash /home/ubuntu/general-mall/health_check.sh

# 服務管理
cd /home/ubuntu/general-mall && docker compose ps
```
