# 環境規格評估

## 測試機

| 項目 | 規格 |
|------|------|
| CPU | 2 核 |
| 記憶體 | 4 GB |
| 硬碟 | 50 GB SSD |
| 頻寬 | 5 Mbps |
| 架構 | 單機 Docker 全包 |

### Docker 服務配置

| 服務 | 記憶體 |
|------|--------|
| MySQL | 512 MB |
| Redis | 256 MB |
| Elasticsearch | 1 GB |
| RabbitMQ | 256 MB |
| mall-admin | 512 MB |
| mall-portal | 512 MB |
| mall-search | 256 MB |
| Nginx | 128 MB |
| **合計** | **~3.5 GB** |

## 正式機

| 項目 | 規格 |
|------|------|
| CPU | 8 核 |
| 記憶體 | 16 GB |
| 硬碟 | 200 GB SSD |
| 頻寬 | 30 Mbps |
| 架構 | EC2 + RDS + ALB |

### AWS 服務

| 服務 | 規格 | 說明 |
|------|------|------|
| EC2 | t3.2xlarge | 應用伺服器 |
| RDS | db.t3.large | Multi-AZ 高可用 |
| ALB | Application LB | HTTPS + 自動分流 |

### Docker 服務配置（正式機）

| 服務 | 記憶體 |
|------|--------|
| Redis | 1 GB |
| Elasticsearch | 4 GB |
| RabbitMQ | 1 GB |
| mall-admin | 2 GB |
| mall-portal | 2 GB |
| mall-search | 1 GB |
| **合計** | **~11 GB** |

## 部署架構圖

```
                    ┌─────────────┐
                    │     ALB     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │     EC2     │
                    │   Docker    │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐      ┌──────────┐     ┌───────────┐
   │   RDS   │      │  Redis   │     │    ES     │
   │ (MySQL) │      │          │     │           │
   └─────────┘      └──────────┘     └───────────┘
```
