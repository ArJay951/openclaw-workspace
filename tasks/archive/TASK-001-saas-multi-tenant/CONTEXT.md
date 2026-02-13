# CONTEXT - 娛樂城商城系統

## 專案路徑

| 元件 | 路徑 |
|------|------|
| 後端 | `/home/ubuntu/entertainment-mall/` |
| 前台前端 | `/home/ubuntu/ent-mall-app-web/` |
| 後台前端 | `/home/ubuntu/ent-mall-admin-web/` |

## 後端結構

```
/home/ubuntu/entertainment-mall/
├── mall-admin/          # 後台 API (port 8090)
│   └── src/main/java/com/macro/mall/
│       ├── config/      # 配置類
│       ├── controller/  # 控制器
│       └── ...
├── mall-portal/         # 前台 API (port 8095)
│   └── src/main/java/com/macro/mall/portal/
│       ├── config/      # 配置類
│       ├── controller/  # 控制器
│       └── ...
├── mall-common/         # 共用模組
│   └── src/main/java/com/macro/mall/common/
├── mall-security/       # 安全模組
├── mall-mbg/           # MyBatis Generator
├── mall-search/        # ES 搜尋
├── pom.xml             # 父 POM
└── document/           # 文件
```

## 技術棧

| 技術 | 版本 |
|------|------|
| Java | 8 |
| Spring Boot | 2.7.x |
| MyBatis | 3.x |
| MySQL | 5.7 (Docker) |
| Redis | 7 (Docker) |
| RabbitMQ | 3 (Docker) |
| Elasticsearch | 7.17 (Docker) |
| Maven | 3.x |

## Docker 容器

| Container | Port | 說明 |
|-----------|------|------|
| mall-mysql | 3306 | MySQL（所有 DB 共用） |
| mall-redis | 6379 | Redis |
| mall-rabbitmq | 5672/15672 | RabbitMQ |
| mall-elasticsearch | 9200/9300 | ES |
| mall-mongo | 27017 | MongoDB |
| ent-mall-admin | 8090 | 娛樂城後台 API |
| ent-mall-portal | 8095 | 娛樂城前台 API |

## 資料庫

- **現有 DB**: `ent_mall`（77 tables）
- **DB 帳號**: root / root
- **charset**: utf8mb4

### 主要表前綴
| 前綴 | 模組 |
|------|------|
| pms_* | 商品 |
| oms_* | 訂單 |
| ums_* | 會員 |
| sms_* | 行銷 |
| cms_* | 內容 |

## 現有配置檔

### 配置檔結構
- `application.yml` — 共用配置（JWT、MyBatis、白名單）
- `application-dev.yml` — 開發環境
- `application-prod.yml` — 生產環境（Docker 內使用）

### mall-portal application-prod.yml
```
路徑: /home/ubuntu/entertainment-mall/mall-portal/src/main/resources/application-prod.yml
  - server.port: 8085（Docker 內部 port，映射到 host 8095）
  - spring.datasource.url: jdbc:mysql://mall-mysql:3306/ent_mall
  - spring.datasource: 使用 Druid 連線池
  - spring.redis.host: mall-redis, database: 1
  - spring.rabbitmq.host: mall-rabbitmq, virtual-host: /ent-mall
  - spring.data.mongodb.host: mall-mongo, database: ent-mall-portal
```

### mall-admin application-prod.yml
```
路徑: /home/ubuntu/entertainment-mall/mall-admin/src/main/resources/application-prod.yml
  - server.port: 8080（Docker 內部 port，映射到 host 8090）
  - 其餘類似 portal
```

### ⚠️ Docker 網路
- 容器在 `mall-network`（external: general-mall_default）
- DB host 用 container name: `mall-mysql`（不是 localhost）
- Redis host: `mall-redis`
- 連線池: Druid（不是 HikariCP）

## Nginx 配置

| 檔案 | 域名 | 指向 |
|------|------|------|
| `/etc/nginx/sites-available/ent-mall` | ent.homely-go.com | dist/ + proxy 8095 |
| `/etc/nginx/sites-available/ent-mall-admin` | ent-admin.homely-go.com | admin dist/ + proxy 8090 |

## 前台前端

```
/home/ubuntu/ent-mall-app-web/
├── src/
│   ├── main.js
│   ├── App.vue         # 全局佈局
│   ├── router/index.js
│   ├── store/index.js  # Vuex state
│   ├── utils/request.js # axios with /portal-api prefix
│   ├── api/            # API 模組
│   └── views/          # 頁面
├── vue.config.js
└── package.json
```

- 技術: Vue 2 + Vant 2 + Vue Router + Vuex
- API prefix: `/portal-api/` → Nginx proxy → localhost:8095

## 後台前端

```
/home/ubuntu/ent-mall-admin-web/
├── src/
│   ├── main.js
│   ├── views/
│   └── ...
└── package.json
```

- 技術: Vue 2 + Element UI
- API prefix: `/admin-api/` → Nginx proxy → localhost:8090

## 部署指令

### 後端 build
```bash
cd /home/ubuntu/entertainment-mall
mvn clean package -DskipTests
```

### 後端重啟
```bash
# portal
docker restart ent-mall-portal
# 或者重新部署
docker stop ent-mall-portal && docker rm ent-mall-portal
docker run -d --name ent-mall-portal \
  -p 8095:8095 \
  --network mall-network \
  -v /home/ubuntu/entertainment-mall/mall-portal/target/mall-portal-1.0-SNAPSHOT.jar:/app.jar \
  openjdk:8-jre \
  java -jar /app.jar

# admin 同理，port 8090
```

### 前端 build
```bash
cd /home/ubuntu/ent-mall-app-web && npm run build
cd /home/ubuntu/ent-mall-admin-web && npm run build
```

## 伺服器
- IP: 52.76.231.27
- OS: Ubuntu Linux
- RAM: 16 GB
- CPU: 4 cores
