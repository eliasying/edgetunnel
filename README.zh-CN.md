# edgetunnel

[English](README.md) | 简体中文

一个运行在 Cloudflare Workers 上的 VLESS-over-WebSocket 代理。单文件 Worker，既可以用 Wrangler 部署，也可以直接粘贴到 Cloudflare 控制台。基于 [zizifn/edgetunnel](https://github.com/zizifn/edgetunnel) 的社区 fork。

## 项目概览

[`src/`](src/) 下有两个自包含的 Worker：

| 文件 | 出口 | DNS（UDP :53） |
|---|---|---|
| `src/worker-vless.js` | `cloudflare:sockets` 直连 TCP，失败时回退 `PROXYIP` | DoH（`1.1.1.1`） |
| `src/worker-with-socks5-experimental.js` | 可选 SOCKS5 上游（`SOCKS5` 环境变量） | TCP 直连 DNS（`8.8.4.4:53`） |

HTTP 行为（两个 Worker 相同）：

| 路径 | 响应 |
|---|---|
| `GET /` | `request.cf` 的 JSON，`colo` 字段即当前服务的边缘机房 |
| `GET /{UUID}` | 可直接粘贴的客户端配置（v2ray/v2rayN 分享 URL + clash-meta YAML） |
| 其他 | `404 Not found` |

WebSocket 升级请求进入 VLESS 隧道处理逻辑。

## ⚠️ 安全警告——部署前必读

- **默认 UUID 硬编码在这个公开仓库里**，任何知道它的人都能把你的 Worker 当开放代理用。请设置自己的 `UUID`（`cat /proc/sys/kernel/random/uuid` 或 PowerShell `[guid]::NewGuid()` 生成）。本 fork 按约定保留了默认值，不接受风险请务必修改。
- `GET /{UUID}` 会向持有 UUID 的人展示完整客户端配置——请把 UUID 当作机密。
- 在 Workers 上自建代理可能违反 [Cloudflare 服务条款](https://www.cloudflare.com/terms/)，账号风险自负。
- Worker 本身没有限流等滥用防护。

## 部署

**方式 A —— Wrangler（推荐）**

```bash
npm install
npm run dev        # 本地开发，端口 8787
npm run deploy     # 按 wrangler.toml 部署 src/worker-vless.js
```

脚本：`dev`（主版本）、`dev-socks5`（实验版）、`deploy`。要部署实验版需把 `wrangler.toml` 的 `main` 指向实验版文件。

**方式 B —— Cloudflare 控制台粘贴**

把 `src/worker-vless.js` 全文复制进新建的 Worker（Workers & Pages → Create → Quick edit），再在 Settings → Variables 里配置下述环境变量。

## 环境变量

| 变量 | 适用 | 含义 | 未设置时的默认 |
|---|---|---|---|
| `UUID` | 两者 | 客户端认证用的 VLESS 用户 ID | 源码中的硬编码值 |
| `PROXYIP` | 两者 | 直连无回包时的重试回落地址（如访问 CF 自家站点） | 空（不回落） |
| `SOCKS5` | 实验版 | 出口走 SOCKS5 服务器：`user:pass@host:port` 或 `host:port`；设置后忽略 `PROXYIP` | 空（禁用） |

## 客户端配置

访问 `https://<你的worker域名>/<UUID>`，复制 `vless://` URL（v2rayN 等）或 clash-meta 片段即可。生成的配置为 TLS + WebSocket，带 0-RTT 早数据（`/?ed=2048`）。

**优选 IP**：想选择更优的入口机房时，只把客户端配置里的服务器地址换成自测良好的 Cloudflare 边缘 IP，WebSocket 的 `Host` 头 / TLS SNI 仍保持 worker 域名不变。入口机房可用 `GET /` 返回的 `colo` 字段验证。

## 区域固定（Placement）

默认情况下 Worker 运行在离访客最近的机房。要固定区域，可设置 Placement（Workers & Pages → Settings → Runtime → Placement，如区域 `azure:westus`），用响应头 `cf-placement` 验证（`remote-SJC` 即远端圣何塞执行）。

注意：`wrangler.toml` 里的 `[placement] region` 需要 Wrangler 4+；本仓库锁定 Wrangler 3，因此 placement 在控制台管理，且重新部署可能重置——每次部署后请复查。

## 本地开发

`config/config-client-with-*-local.json` 是配合 `wrangler dev` 使用的 Xray 客户端示例（SOCKS :4080 / HTTP :4081 入站）。`test/` 是原作者的独立实验脚本，原样保留。

## 致谢

- [zizifn/edgetunnel](https://github.com/zizifn/edgetunnel) —— 本 fork 所基于的原始项目。

## 许可证

GPL-2.0，见 [LICENSE](LICENSE)。fork 需保持源码开放。
