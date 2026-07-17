# Blackcat Companion Prototype

这个仓库包含 P0 产品文档、原型演示和正在开发的最小业务系统代码。

## 文档站点

GitHub Pages 发布目录：`docs/`

入口页面：`docs/index.html`

## 本地开发

要求：

- Node.js 22+
- npm 10+
- Docker Desktop，用于本地 PostgreSQL 与 Compose smoke

常用命令：

```bash
npm install
docker compose up -d postgres
npm run db:migrate:deploy
npm run dev
npm run m0:verify
npm run typecheck
docker compose config
```

`npm run dev` 会使用 `.env.example` 启动 API、Sapphire Bot 适配器和 Dashboard。当前 Discord Bot credential 未提供时，Bot 不登录 Discord；这不阻断 M0 本地骨架验证。

数据库连接分两类：应用进程使用 `DATABASE_URL` 中的 `blackcat_app` 角色；迁移使用 `MIGRATION_DATABASE_URL` 中的 owner 角色。`GET /ready` 会使用应用角色登录并检查 baseline schema，因此 fresh PostgreSQL 必须先执行 `npm run db:migrate:deploy`。

API 本地端点：

- `GET /health`：进程存活检查，对应 `getHealth`
- `GET /ready`：依赖就绪检查，对应 `getReadiness`

开发规则见 [AGENTS.md](/Users/kulbear/Documents/Codex/2026-07-15/referenced-chatgpt-conversation-this-is-untrusted/AGENTS.md)。
