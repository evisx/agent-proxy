# agent-proxy

`agent-proxy` 是一个基于 TypeScript 和 Cloudflare Workers 的无状态 HTTP/HTTPS relay 代理。它仅处理携带共享 secret 的内部 relay 路径，并将请求尽量透明地转发到目标上游。

该项目的目标很明确：提供一个职责单一、部署轻量、便于组合到现有分发链路中的 relay 节点。它不负责鉴权面板、存储、统计、绕过挑战或其他与“透明转发”无关的能力。

## 核心能力

- 基于路径表达目标上游，无需额外编码协议
- 同时支持 HTTP 与 HTTPS 上游转发
- 尽量保留原始请求方法、查询参数、请求头、响应头与流式 body
- 通过共享 `DISPATCH_SECRET` 限制内部 relay 路径访问
- 自动阻断明显的自代理与循环代理场景

## 部署

### 一键部署到 Cloudflare Workers

部署前请先 `fork` 本项目到你自己的 GitHub 账号。

1. `fork` 本项目到你的 GitHub 账号，例如：`https://github.com/YOUR_GITHUB_USERNAME/agent-proxy`
2. 将下面链接中的 `YOUR_GITHUB_USERNAME` 替换为你的 GitHub 用户名后再访问：

```text
https://deploy.workers.cloudflare.com/?url=https://github.com/YOUR_GITHUB_USERNAME/agent-proxy
```

示例：

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/YOUR_GITHUB_USERNAME/agent-proxy)

## 快速开始

```bash
npm install
npm run dev
```

常用命令：

- `npm run dev`：本地启动 Worker
- `npm run typecheck`：执行 TypeScript 类型检查
- `npm test`：运行单元与集成测试
- `npm run build`：执行 Wrangler dry-run 构建

## 路由规则

### HTTP relay

请求：

```text
GET /relay/<DISPATCH_SECRET>/proxy/www.google.com/search?q=workers
```

转发到：

```text
http://www.google.com/search?q=workers
```

### HTTPS relay

请求：

```text
GET /relay/<DISPATCH_SECRET>/proxyssl/github.com/repos/cloudflare/workers?tab=actions
```

转发到：

```text
https://github.com/repos/cloudflare/workers?tab=actions
```

### 约束说明

- 只有 `/relay/<DISPATCH_SECRET>/proxy...` 与 `/relay/<DISPATCH_SECRET>/proxyssl...` 会进入代理主流程
- `DISPATCH_SECRET` 不匹配时统一返回 `404`
- `<site>` 必须是合法的上游 `authority`，即 `host` 或 `host:port`
- `<site>` 之后的 path 与 query 会按原样拼接回目标 URL
- `Authorization` 等端到端请求头会透传，但不会参与上游地址解析

## 运行时配置

通过 Worker 环境变量控制：

- `ROUTE_BASE_PATH`
  默认值：空字符串
  用途：为 relay 路由增加公共前缀，例如 `/edge/relay/<DISPATCH_SECRET>/proxyssl/example.com`
- `SELF_HOSTNAMES`
  默认值：空字符串
  用途：逗号分隔的自代理主机名或 self-origin，用于阻断明显的循环代理
- `DISPATCH_SECRET`
  默认值：空字符串
  用途：限制内部 relay 路径访问；为空时所有 relay 请求都会返回 `404`

即使 `SELF_HOSTNAMES` 为空，Worker 仍会自动阻止代理回当前请求自身的 `hostname` 或 `host`。

## 透明转发边界

请求侧会尽量保留：

- HTTP method
- query string
- `User-Agent`
- `Cookie`
- 请求 body，包括 JSON、表单和二进制内容
- 端到端请求头

响应侧会尽量保留：

- 状态码
- 响应头
- `Set-Cookie`
- 流式响应 body，例如 SSE

为了维持标准代理语义，Worker 会主动移除不应继续转发的 hop-by-hop 头部，例如 `Connection`、`Transfer-Encoding`、`Host`。

## 对外暴露边界

- 本项目不是面向终端用户的公网应用入口
- 更推荐与上游分发层组合使用，例如先进入本地或内网中的 `agent-dispatch`
- 旧版直接对外暴露的 `/proxy`、`/proxyssl` 路径不再兼容，当前会统一返回 `404`

## 非目标范围

本项目刻意不提供以下能力：

- 数据库、KV、缓存或其他外部存储
- 管理后台、登录面板、密钥托管、请求统计
- Cloudflare 或其他挑战绕过
- CAPTCHA 求解
- `cf_clearance` 托管
- User-Agent 自动伪装或轮换
- TLS 或 JA3 指纹模拟
- Cookie Jar 持久化
- WebSocket、CONNECT、SOCKS 或系统代理协商

如果上游返回 `403`、`429`、挑战页或其他反爬响应，Worker 会原样透传，不会尝试规避、掩盖或自动重试。

## 已知限制

- relay secret 位于 URL path 中，部署时应避免在日志、监控或链路追踪中记录完整 relay URL
- 当前实现不包含自动重试、健康检查或故障切换
- 某些目标相关头部可能在运行时被接管，因此“透明”表示尽量保持语义一致，而非逐字节完全一致
- `Set-Cookie` 是否最终被浏览器接受，仍取决于上游域名、`Secure`、`SameSite` 等浏览器规则
- 项目默认不提供 allowlist、denylist 或更细粒度访问控制；如需进一步收口，应在部署层额外补充保护

## 使用说明与免责声明

- 本项目主要面向个人学习、研究与技术交流场景
- 使用者在部署、修改或集成本项目时，应自行确认其用途符合适用法律法规、目标平台政策以及所在组织的安全与合规要求
- 作者与维护者不对因使用、误用或二次分发本项目产生的直接或间接损失、合规风险或第三方争议承担责任
- 本节为使用说明，具体授权范围仍以项目许可证为准

## License

本项目采用 `GPL-3.0` 许可证。详情见 [LICENSE](./LICENSE)。
