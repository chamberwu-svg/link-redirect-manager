# Link Redirect Manager

一个基于 Bun 和 PostgreSQL 的子链接入口重定向管理系统，支持同域名下 IP 固定分配、按国家代码限制、访问日志和 Web 管理面板。

## 功能特性

- IP 固定分配：同一域名下，首次访问的 IP 会随机绑定到一个子链接，后续访问保持不变
- 地区限制：按域名维护封禁国家代码列表
- 多域名管理：每个域名拥有独立的子链接池、地区限制、IP 固定分配和访问日志
- 访问记录：记录分配创建、分配复用、无可用链接、国家拦截等事件
- Web 管理面板：在浏览器中直接创建入口域名、添加子链接、管理国家限制并查看日志

## 环境要求

- Bun
- PostgreSQL
- DATABASE_URL 环境变量
- Cloudflare / Railway 二选一或同时配置（用于自动绑定入口域名，可选）
- 管理后台使用 Basic Auth（用户名 + 密码）
- 内置账号：
	- admin（密码来自 `ADMIN_PASSWORD`，默认 `xiaozhangnb`）
	- qilongzhu（密码来自 `QILONGZHU_PASSWORD`，默认 `qilongzhu888`）
- 多账号完全数据隔离：每个账号只能看到和操作自己创建的域名、链接、IP 分配、国家限制和访问日志

Cloudflare Token 相关环境变量：

- CLOUDFLARE_API_TOKEN
- CLOUDFLARE_API_BASE（可选，默认 `https://api.cloudflare.com/client/v4`）
- CLOUDFLARE_ZONE_ID（可选，手动固定到单个 zone 的覆盖值；多域名场景建议留空，让系统按域名自动查找 zone）
- CLOUDFLARE_DNS_TARGET（可选，自动 CNAME 指向目标，如 `your-service.up.railway.app`）
- CLOUDFLARE_DNS_PROXIED（可选，默认 `true`）
- QILONGZHU_PASSWORD（可选，第二个内置账号 `qilongzhu` 的密码，默认 `qilongzhu888`）

Railway 自定义域名自动绑定相关环境变量（推荐配置）：

- RAILWAY_TOKEN（Railway API Token）
- RAILWAY_PROJECT_ID
- RAILWAY_ENVIRONMENT_ID
- RAILWAY_SERVICE_ID
- RAILWAY_APP_DOMAIN（可选，仅用于配置提示）

当配置 `CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_DNS_TARGET` 时：

- 创建入口域名会自动按域名名称查找对应的 Cloudflare Zone，并创建或更新 CNAME 记录

当同时配置 Railway 变量时：

- 创建入口域名会先调用 Railway GraphQL API 绑定自定义域名，再同步 Cloudflare DNS

创建入口域名时支持 `provision_channel`（可在管理面板选择）：

- `auto`（默认）：按已配置项自动尝试 Railway/Cloudflare，任一失败不会回滚域名记录
- `cloudflare`：仅走 Cloudflare，失败会回滚新建域名
- `railway`：仅走 Railway，失败会回滚新建域名
- `both`：Railway + Cloudflare 都必须成功，任一失败会回滚新建域名
- `manual`：只创建系统记录，不做自动绑定

## 快速开始

### 本地开发

```bash
bun install
export DATABASE_URL="postgres://user:password@localhost:5432/link_redirect_manager"
bun run dev
```

启动后访问 http://localhost:8000/admin 进入管理面板。

## 主要接口

- GET /admin：管理面板
- GET /api/overview：面板概览数据
- POST /api/domains：创建域名
- DELETE /api/domains/:id：删除域名
- GET /api/links?domain_id=:id：获取某个域名下的链接
- POST /api/links：新增子链接
- DELETE /api/links/:id：删除子链接
- GET /api/blocked-countries?domain_id=:id：获取某域名的国家限制
- POST /api/blocked-countries：添加国家限制
- DELETE /api/blocked-countries/:countryCode：删除国家限制，需通过请求头 X-Domain-Id 传入域名 ID
- GET /api/assignments?domain_id=:id：查看 IP 固定分配记录
- GET /api/access-logs?domain_id=:id：查看访问日志
- GET /api/redirect/:domainName：执行 IP 锁定重定向解析
- GET /api/version：查看当前运行版本（用于确认部署是否最新）
- GET /healthz：健康检查（不依赖数据库，可用于平台健康探针）
- GET /api/cloudflare/token/status：查看 Cloudflare API Token 配置和校验状态

另外，服务现在支持“按访问 Host 自动重定向”：

- 当请求不是 /api 路径，且请求 Host 与数据库中某个 domain_name 匹配时，会直接返回 302 到分配后的子链接
- 这意味着你购买的域名应当接入这个跳转服务，而不是绑定到前端站点

示例：

1. 在管理面板中创建域名 `example.com`
2. 给 `example.com` 配置多个子链接
3. 将 DNS 指向部署地址（A/CNAME 到你的服务）
4. 用户访问 `https://example.com` 时，会按 IP 锁定规则跳转到对应子链接

`POST /api/domains` 请求体示例：

```json
{
	"domain_name": "go.example.com",
	"provision_channel": "auto"
}
```

## 部署到 Railway

1. 连接仓库到 Railway
2. 添加 PostgreSQL 服务并注入 DATABASE_URL
3. 使用仓库中的 Dockerfile 或 railway.json 启动

## 技术栈

- Bun + TypeScript
- PostgreSQL
- 原生 HTTP API + 内嵌管理面板

## 许可证

MIT
