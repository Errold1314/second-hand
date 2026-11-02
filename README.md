# 校园二手经济 · 本地账号基础

React + TypeScript + Vite / NestJS + TypeScript / PostgreSQL + Prisma。当前包含注册、登录、个人中心、退出与真实修改密码；不含商品、支付、真实短信、微信授权、实名或学生认证。网页名称尚未确定，界面暂用中性账号文案。

**这是开发阶段的手机号 + 密码方案。手机号尚未验证，仅作登录标识。已在 Docker Linux 容器运行正式迁移、真实 PostgreSQL 集成测试及浏览器注册登录，并验证 API/数据库重启后用户与会话保留。仅完成本地账号验收，不代表生产安全或合规验收。** 详见 [验证记录](docs/TESTING.md)。

## 目录

```text
apps/web/               React 页面、API 客户端、前端测试
apps/api/src/auth/      注册登录、CSRF、会话、服务端登录守卫
apps/api/src/users/     用户查询与公开响应字段
apps/api/src/database/  Prisma 生命周期、数据库就绪重试
apps/api/prisma/        数据模型与版本化 SQL 迁移
apps/api/test/          安全单元测试、真实数据库集成测试
apps/api/scripts/       安全查询最近用户（手机号脱敏）
docker/                 Nginx 同源代理、数据库专用用户初始化
scripts/init-env.ts     生成随机本地秘密，不覆盖已有 .env
docs/                   接口、测试与上线限制
compose.yaml            本地完整容器运行
compose.dev.yaml        可选数据库回环端口
```

不需要共享代码时不创建空的 packages。原方案文档和已有 MCP/编辑器文件未改动。本次 TypeScript 后端替代原文中待评估的 Java/若依路线，PostgreSQL 与模块化单体保持不变。

## 环境

- Node.js 24.19.x / npm 11.x；Linux 上 Docker Engine 与 Compose v2；Windows 可使用已有 Docker Desktop Linux 容器环境。
- 精确版本与完整依赖树见 package.json 与 package-lock.json；使用 npm ci，不将 Prisma 自动升级到预发布版本。
- 不需要 GitHub MCP、短信账号或真实学生数据。无默认管理员。

## 一键本地容器启动

在项目根目录执行：

```sh
npm run setup:env
npm run up
```

第一条需 Node24，生成未跟踪的 .env，秘密不会输出，已有文件不会覆盖。随后 Compose 构建前后端，首次初始化新数据卷的数据库及专用应用用户，数据库健康后运行 `prisma migrate deploy`，迁移成功才启动 API，最后启动网页。

如果自行填写配置，参考 [.env.example](.env.example)。不要将真实秘密提交仓库；Windows 需核对 .env 文件 ACL。修改已有数据卷的环境密码不会自动修改数据库角色密码，请按数据库管理流程轮换，不要删除数据卷解决密码问题。

当前本机容器版使用 5174（5173 被旧 Vite 进程占用），本地 .env 已同步设置 WEB_PORT=5174、WEB_ORIGIN=http://localhost:5174：
- 注册：http://localhost:5174/register
- 登录：http://localhost:5174/login
- 个人中心：http://localhost:5174/me
- 健康：http://localhost:5174/api/health

新环境默认仍是5173；若修改端口，必须同时修改 WEB_PORT 和 WEB_ORIGIN。原生 Vite 默认5173，切回原生开发时也应同步 WEB_ORIGIN。

Windows 安装 Docker 后若旧 VS Code 终端提示找不到命令，重启 VS Code，或仅在当前 PowerShell 设置 PATH：

```powershell
$env:PATH = "$env:LOCALAPPDATA\Programs\DockerDesktop\resources\bin;" + $env:PATH
```

```sh
npm run logs                 # 最近100行并持续跟踪
npm run down                 # 停止并移除容器/网络，保留数据卷
npm run up                   # 重新启动，保留用户
```

**不要执行 `docker compose down -v`、volume prune 或数据库重置命令。** 数据卷不等于备份。

数据库和 API 默认不发布宿主端口，网页仅绑定 127.0.0.1。Compose 是本地 HTTP 开发配置，明确 NODE_ENV=development，不可直接公网部署。镜像使用 node:24.19.0-bookworm（API）、node:24.19.0-bookworm-slim（前端构建）、postgres:17.10-bookworm、nginx:1.28.3-alpine，未使用 latest。

### 首次迁移与后续迁移

首次 `npm run up` 已包含独立 migrate 服务；查看：

```sh
docker compose logs migrate
# 修改已有系统结构前先备份，再单独执行受控迁移：
docker compose run --rm migrate
```

不使用 db push 或普通启动重置数据库。SQL 位于 apps/api/prisma/migrations；数据库上的 _prisma_migrations 记录执行情况。应用用户 campus_app 非超级用户，不具备创建数据库/角色权限；首版为迁移拥有 public schema 创建权限。生产应进一步分离迁移账户与运行账户。

## 原生前后端开发 + Docker 数据库

```sh
npm ci
npm run setup:env
npm run db:generate
docker compose -f compose.yaml -f compose.dev.yaml up -d db
npm run db:migrate
npm run build -w @campus/api
npm run dev:api
```

另开一个终端：

```sh
npm run dev:web
```

开发前端默认只监听 127.0.0.1:5173，后端只监听 127.0.0.1:3000。后端 dev 模式监视编译输出，改后端源码后另运行 `npm run build -w @campus/api`。终端 Ctrl+C 停止前后端；数据库用 Compose down，保留卷。

可选开发配置只将数据库映射至 127.0.0.1:5432。若端口被占用，请调整覆盖文件与 .env 的 DATABASE_URL，不要停止不属于本项目的数据库。DATABASE_URL 仅用于后端，绝不添加 VITE_ 前缀。前端经 Vite /api 代理请求，不启用跨域。

## 测试与安全查询用户

```sh
npm run db:generate
npm run typecheck
npm run build
npm test
npm audit
npm run users:list -w @campus/api
```

最后一条在原生数据库回环配置下运行，只查询最近20个用户的 ID、脱敏手机号、状态和创建时间，不输出密码/会话哈希。容器模式可用只读 SQL：

```sh
docker compose exec db psql -U campus_owner -d campus -c "SELECT id, left(phone,6) || '****' || right(phone,4) AS phone_masked, status, created_at FROM users ORDER BY created_at DESC LIMIT 20;"
```

真实数据库集成测试需独立测试数据库及明确授权开关，详见 [测试步骤](docs/TESTING.md)，不得用真实学生资料测试。

## 安全边界与后续上线

- Argon2id 哈希密码（64MiB、3轮、独立随机盐）。随机32字节会话令牌只在 HttpOnly Cookie，数据库保存 SHA-256 哈希；有效期、撤销和账号状态每次服务端检查。
- CSRF HMAC签名双提交令牌 + 精确 Origin；同源开发代理；生产模式 Cookie Secure，生产来源必须 HTTPS。
- 严格输入字段、唯一约束与注册事务、基本内存限流、参数化查询、不记录敏感请求、认证响应 no-store。
- 已登录修改密码需校验原密码；新哈希与全部设备会话撤销事务提交，成功后重新登录。没有用短信绕过原密码的开发后门。
- 短信验证码、微信授权、手机号绑定与验证码找回仅展示明确未接通的入口。真实启用需供应商账号/签名模板、微信官方应用及回调配置；微信不会自动返回绑定手机号，不能凭前端信息标记验证成功。
- **手机号目前在数据库中规范化明文存储，未实现字段加密**；会话过期清理、账号找回、共享限流和管理员体系不在本次范围。
- 上线前必须补齐真实短信归属验证、号码回收/换绑恢复规则、HTTPS及生产配置、可信代理及多维限流、秘密管理、数据库/备份加密与异机恢复、监控告警、依赖维护及独立安全验收。
- 按实际业务办理主体、备案/许可，完善隐私政策、用户协议、注销和数据保存规则；核实学生认证、未成年人及个人信息合规要求。不因没有支付就认定无需相关手续。

[接口与安全设计](docs/API.md) · [实际测试结果和未完成项](docs/TESTING.md) · [后续修改代码入口](docs/CODE_GUIDE.md) · [账号 UI 与短信/微信接入边界](docs/ACCOUNT_UI.md)
