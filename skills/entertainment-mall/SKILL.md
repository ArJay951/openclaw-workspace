---
name: entertainment-mall
description: 娱乐城积分兑换商城系统开发指南。基于 macrozheng/mall 框架，对接娱乐城平台 API，支持多套独立部署。用于：(1) 商城系统开发与维护 (2) 功能需求讨论 (3) 部署与配置 (4) 娱乐城 API 对接
---

# 娱乐城商城系统 (Entertainment Mall)

## 项目概述

独立的积分兑换电商平台，提供 API 对接娱乐城平台。支持多套独立部署，每套服务一个娱乐城。

## 技术架构

### 后端
- **框架**: Spring Boot + MyBatis
- **数据库**: MySQL
- **缓存**: Redis
- **搜索**: Elasticsearch
- **消息队列**: RabbitMQ
- **部署**: Docker 容器化

### 前端
- **后台管理**: Vue + Element UI
- **前台商城**: uni-app（样式参考 lhshop.online）

## 核心模块

| 模块 | 来源 | 说明 |
|------|------|------|
| mall-admin | mall | 后台管理 API |
| mall-portal | mall | 前台商城 API |
| mall-common | mall | 通用工具 |
| mall-security | mall (改造) | 安全认证，对接娱乐城 |
| mall-mbg | mall | MyBatis 代码生成 |
| mall-search | mall | Elasticsearch 搜索 |

## 需要改造的部分

### 1. 用户认证
- 原: 手机号/邮箱注册登录
- 改: 对接娱乐城 API 认证

### 2. 支付系统
- 原: 支付宝/微信支付
- 改: 点数扣除（调用娱乐城 API）

### 3. 虚拟商品
- 新增: 彩金兑换 → 调用娱乐城 API 发放余额

### 4. 多租户配置
- 新增: 商城名称、Logo、API 端点可配置

## 部署规范

### 环境分离
- **测试环境**: 可自由更新部署
- **正式环境**: 🔒 需授权才能更新

### Docker 部署
每套商城独立容器，包含：
- 应用服务 (mall-admin, mall-portal)
- MySQL
- Redis
- Elasticsearch
- RabbitMQ
- Nginx

## 参考文档

- [需求边界](references/requirements.md) - 完整需求规格
- [技术架构](references/architecture.md) - 详细架构设计
- [数据库设计](references/database.md) - 数据库表结构
- [API 规格](references/api-spec.md) - 娱乐城对接 API

## 源码位置

- mall 源码: `/home/ubuntu/mall-source`
- 数据库脚本: `/home/ubuntu/mall-source/document/sql/mall.sql`
