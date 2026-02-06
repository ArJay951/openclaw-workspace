# 技术架构文档

## 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (反向代理)                       │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  mall-admin   │    │  mall-portal  │    │  mall-search  │
│  (后台 API)    │    │  (前台 API)    │    │  (搜索服务)    │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│     MySQL     │    │     Redis     │    │ Elasticsearch │
│   (主数据库)   │    │    (缓存)     │    │    (搜索)     │
└───────────────┘    └───────────────┘    └───────────────┘
                              │
                              ▼
                    ┌───────────────┐
                    │   RabbitMQ    │
                    │  (消息队列)    │
                    └───────────────┘
                              │
                              ▼
                    ┌───────────────┐
                    │  娱乐城 API   │
                    │  (外部对接)    │
                    └───────────────┘
```

## 模块说明

### mall-admin (后台管理)
- 端口: 8080
- 功能: 商品、订单、会员、促销、内容、权限管理
- 认证: JWT Token

### mall-portal (前台商城)
- 端口: 8085
- 功能: 商品浏览、购物车、订单、会员中心
- 认证: JWT Token（对接娱乐城）

### mall-search (搜索服务)
- 端口: 8081
- 功能: 商品搜索、商品推荐
- 依赖: Elasticsearch

### mall-security (安全模块)
- 共用模块，封装 Spring Security
- 需改造: 对接娱乐城用户认证

### mall-common (通用模块)
- 工具类、通用代码
- API 响应封装

### mall-mbg (代码生成)
- MyBatis Generator
- 生成数据库操作代码

## 技术栈版本

| 组件 | 版本 |
|------|------|
| JDK | 1.8 / 17 (dev-v3 分支) |
| Spring Boot | 2.7.x / 3.2.x |
| MySQL | 5.7+ |
| Redis | 7.0 |
| Elasticsearch | 7.17.3 |
| RabbitMQ | 3.10.5 |
| Nginx | 1.22 |

## Docker 部署架构

```yaml
# docker-compose.yml 结构
services:
  mall-admin:
    image: mall/mall-admin
    ports: ["8080:8080"]
    
  mall-portal:
    image: mall/mall-portal
    ports: ["8085:8085"]
    
  mall-search:
    image: mall/mall-search
    ports: ["8081:8081"]
    
  mysql:
    image: mysql:5.7
    ports: ["3306:3306"]
    
  redis:
    image: redis:7
    ports: ["6379:6379"]
    
  elasticsearch:
    image: elasticsearch:7.17.3
    ports: ["9200:9200"]
    
  rabbitmq:
    image: rabbitmq:3.10-management
    ports: ["5672:5672", "15672:15672"]
    
  nginx:
    image: nginx:1.22
    ports: ["80:80", "443:443"]
```

## 多租户架构

每套商城独立部署，通过环境变量配置：

```bash
# 商城配置
MALL_NAME=隆亨精品商城
MALL_LOGO_URL=https://...

# 娱乐城 API
CASINO_API_URL=https://api.casino.com
CASINO_API_KEY=xxx

# 数据库
MYSQL_HOST=mysql
MYSQL_DATABASE=mall_lh

# 其他服务
REDIS_HOST=redis
ES_HOST=elasticsearch
```

## 环境分离

### 测试环境
- 域名: test.mall.example.com
- 可自由部署更新
- 使用测试数据库

### 正式环境
- 域名: mall.example.com
- 🔒 需授权才能更新
- 使用生产数据库
- 定期备份
