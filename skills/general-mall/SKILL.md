---
name: general-mall
description: 通用商城系統開發指南。基於 macrozheng/mall 框架，保留完整功能，支援多套獨立部署。用於：(1) 商城系統部署與維護 (2) 多租戶配置 (3) AWS 環境部署
---

# 商城系統 (General Mall)

## 專案概述

一般電商平台，基於 mall 完整功能，支援多套獨立部署。

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

### 新增功能：自動產生訂單 API

提供 API 讓下游系統傳入金額，自動組合商品產生訂單。

詳見：[自動產生訂單 API](references/auto-order-api.md)

## 多租戶配置

每套商城可獨立配置：
- 商城名稱
- 商城 Logo
- 資料庫連接
- 其他品牌設定

## 部署架構

- **測試機**: 單機 Docker 全包
- **正式機**: EC2 + RDS (Multi-AZ) + ALB

## 參考文件

- [需求邊界](references/requirements.md)
- [環境規格](references/environment.md)
- [費用評估](references/cost.md)
- [自動產生訂單 API](references/auto-order-api.md)

## 源碼位置

- mall 源碼: `/home/ubuntu/mall-source`
