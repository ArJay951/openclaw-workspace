---
name: mall-dev
description: 隆亨精品商城二开项目开发助手。用于：(1) 查询项目结构和上下文 (2) 执行构建/运行/测试 (3) 同步进度到 Telegram (4) 管理外部 API 接口设计。当涉及 mall 项目、隆亨商城、点数兑换系统开发时使用。
---

# Mall 项目开发 Skill

## 快速上下文

```
项目：隆亨精品商城（基于 macrozheng/mall 二开）
分支：dev-v3 (Spring Boot 3.2 + JDK 17)
架构：单体
前端：Vue 2 + Element UI
核心改动：金额支付 → 点数兑换
```

## 项目路径

- **后端:** `/home/ubuntu/.openclaw/workspace/mall/`
- **前端后台:** `/home/ubuntu/.openclaw/workspace/mall-admin-web/`
- **前端商城:** `/home/ubuntu/.openclaw/workspace/mall-app-web/`
- **规划文档:** `mall/docs/PROJECT_PLAN.md`
- **项目上下文:** 读取 `references/project-context.md`
- **API 规格:** 读取 `references/api-spec.md`

## 常用命令

```bash
# === 后端 ===
# 编译（跳过测试）
cd /home/ubuntu/.openclaw/workspace/mall && mvn clean install -DskipTests

# 运行 admin API
cd /home/ubuntu/.openclaw/workspace/mall/mall-admin && mvn spring-boot:run

# 运行 portal API
cd /home/ubuntu/.openclaw/workspace/mall/mall-portal && mvn spring-boot:run

# === 前端后台 ===
cd /home/ubuntu/.openclaw/workspace/mall-admin-web && npm install && npm run dev

# === 前端商城 ===
cd /home/ubuntu/.openclaw/workspace/mall-app-web && npm install && npm run dev
```

## 进度同步

同步消息到 Telegram（阿杰 chat ID: 7268541653）：

```
message action=send channel=telegram target=7268541653 message="<进度内容>"
```

## 外部 API 预留

三个核心外部接口（详见 `references/api-spec.md`）：

| 服务 | 用途 | 状态 |
|------|------|------|
| ExternalAuthService | 登录/注册 | 待对接 |
| ExternalPointsService | 点数余额/充值 | 待对接 |
| ExternalExchangeService | 商品兑换（扣点）| 待对接 |

## 开发阶段

| Phase | 内容 | 状态 |
|-------|------|------|
| 1 | 环境 + DB 改造 + 接口层骨架 | 进行中 |
| 2 | 后端业务改造 | 待开始 |
| 3 | 前端重构 | 待开始 |
| 4 | 联调测试 | 待开始 |

## 注意事项

- 修改代码后 commit 到本地
- 重要进度同步到 Telegram
- 外部 API 先用 Mock 实现
- 最小化改动原有 mall 代码，便于后续升级
