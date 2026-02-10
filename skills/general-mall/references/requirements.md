# 需求邊界文件

## 專案定位

| 項目 | 內容 |
|------|------|
| **類型** | 一般電商平台 |
| **部署模式** | 多套獨立部署 |
| **商城名稱** | 可自定義配置 |

## 技術選型

| 項目 | 內容 |
|------|------|
| **參考源碼** | macrozheng/mall |
| **後端** | Spring Boot + MyBatis + MySQL + Redis + ES + RabbitMQ |
| **前台框架** | uni-app（原版） |
| **後台框架** | Vue + Element UI |
| **部署** | Docker + AWS |

## 功能範圍

完整保留 mall 功能：

### 前台功能
- 首頁門戶
- 商品推薦
- 商品搜尋
- 商品展示
- 購物車
- 訂單流程
- 會員中心
- 客戶服務
- 幫助中心

### 後台功能
- 商品管理
- 訂單管理
- 會員管理
- 促銷管理
- 運營管理
- 內容管理
- 統計報表
- 財務管理
- 權限管理

## 支付

暫不對接，保留原版支付接口結構，未來可擴展。

## Auto-Order API（代收代付）

### 功能需求

提供 API 讓下游系統（如支付網關）自動產生訂單：

| 類型 | 說明 |
|------|------|
| **代收 (collect)** | 創建購買訂單，用於收款 |
| **代付 (pay)** | 創建退貨訂單，用於付款 |

### API 端點

```
POST /order/auto-generate          # 通用端點（需傳 type）
POST /order/auto-generate/collect  # 代收專用
POST /order/auto-generate/pay      # 代付專用
```

### 請求參數

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| type | string | ✅ | `collect` 或 `pay` |
| amount | number | ✅ | 訂單金額 |
| name | string | ✅ | 收件人/退貨人姓名 |
| phone | string | ✅ | 手機號碼 |
| source | string | | 來源標識（自訂） |
| sourceDomain | string | | 來源網域（手動指定） |

### 來源 IP 白名單管理

**需求**：驗證請求來源 IP，只允許白名單內的 IP 調用 API。

#### 功能說明

1. **IP 自動獲取**（後端）
   - 從請求中取得真實 IP（考慮反向代理）
   - 優先順序：`X-Forwarded-For` → `X-Real-IP` → `RemoteAddr`
   - 不接受前端傳入，避免篡改

2. **來源管理表**
   - 來源名稱（如 casino-a）
   - IP 白名單（支援多個 IP，支援 CIDR 如 `203.1.2.0/24`）
   - 狀態：啟用/停用

3. **後台管理介面**
   - 新增選單：「來源管理」
   - 功能：新增/編輯/刪除來源、設定 IP 白名單、啟用/停用

4. **API 驗證邏輯**
   - 請求 → 取得 IP → 查表匹配 → 記錄來源名稱到訂單
   - **IP 不在白名單 → 拒絕請求**

#### 資料庫結構

```sql
-- 來源管理表
CREATE TABLE oms_order_source (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL COMMENT '來源名稱',
  ip_whitelist TEXT COMMENT 'IP白名單，逗號分隔，支援CIDR',
  status INT DEFAULT 1 COMMENT '0=停用 1=啟用',
  create_time DATETIME,
  update_time DATETIME
);

-- 訂單表擴展
ALTER TABLE oms_order 
  ADD COLUMN source_ip VARCHAR(50) COMMENT '來源IP',
  ADD COLUMN source_name VARCHAR(50) COMMENT '來源名稱';
```

### 商品組合演算法

1. 取得所有上架商品，按價格降序排列
2. 貪心演算法：優先選高價商品填滿金額
3. 剩餘金額 < 最低價商品 → 產生折扣/調整
4. 金額不足最低價商品 → 返回錯誤

### 安全設定

- API 端點需加入安全白名單（免 JWT 驗證）
- 可擴展：來源網域白名單驗證（限制特定網域調用）

## 可配置項（多套部署）

每套商城可獨立配置：
- 商城名稱
- 商城 Logo
- 資料庫連接
- Redis 連接
- ES 連接
- 其他品牌相關設定
