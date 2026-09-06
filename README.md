# edgetunnel

English | [简体中文](README.zh-CN.md)

A VLESS-over-WebSocket proxy running on Cloudflare Workers. Single-file workers you can deploy with Wrangler or paste straight into the Cloudflare dashboard. Community fork of [zizifn/edgetunnel](https://github.com/zizifn/edgetunnel).

## Overview

Two self-contained workers live in [`src/`](src/):

| File | Exit | DNS (UDP :53) |
|---|---|---|
| `src/worker-vless.js` | Direct TCP via `cloudflare:sockets`; falls back to `PROXYIP` on retry | DNS-over-HTTPS (`1.1.1.1`) |
| `src/worker-with-socks5-experimental.js` | Optional SOCKS5 upstream (`SOCKS5` env) | Plain DNS over TCP (`8.8.4.4:53`) |

HTTP behavior (both workers):

| Path | Response |
|---|---|
| `GET /` | JSON of `request.cf` — shows the edge datacenter serving you (`colo` field) |
| `GET /{UUID}` | Ready-to-paste client config (v2ray/v2rayN share URL + clash-meta YAML) |
| anything else | `404 Not found` |

WebSocket upgrade requests enter the VLESS tunnel handler.

## ⚠️ Security warnings — read before deploying

- **The default UUID is hard-coded in this public repository.** Anyone who knows it can use your deployed Worker as an open proxy. Set your own `UUID` (e.g. generate one with `cat /proc/sys/kernel/random/uuid` or PowerShell `[guid]::NewGuid()`). This fork intentionally keeps the default value; change it unless you accept the risk.
- `GET /{UUID}` reveals the full client config to anyone holding the UUID — treat the UUID as a secret.
- Self-hosting a proxy on Workers may violate the [Cloudflare Terms of Service](https://www.cloudflare.com/terms/); your account is your own responsibility.
- The worker itself does no rate limiting or abuse controls.

## Deploy

**Option A — Wrangler (recommended)**

```bash
npm install
npm run dev        # local dev server on :8787
npm run deploy     # deploy src/worker-vless.js per wrangler.toml
```

Scripts: `dev` (main worker), `dev-socks5` (experimental worker), `deploy`. Deploying the experimental worker instead requires pointing `main` in `wrangler.toml` at it.

**Option B — Cloudflare dashboard paste**

Copy the full content of `src/worker-vless.js` into a new Worker (Workers & Pages → Create → Quick edit), then set the environment variables below in Settings → Variables.

## Environment variables

| Variable | Worker | Meaning | Default when unset |
|---|---|---|---|
| `UUID` | both | VLESS user ID for client authentication | hard-coded value in source |
| `PROXYIP` | both | Fallback address when a direct TCP connection returns no data (e.g. for Cloudflare-hosted destinations) | empty (no fallback) |
| `SOCKS5` | experimental | Route exit traffic through a SOCKS5 server: `user:pass@host:port` or `host:port`. Setting it ignores `PROXYIP` | empty (disabled) |

## Client configuration

Visit `https://<your-worker-domain>/<UUID>` and copy either the `vless://` URL (v2rayN et al.) or the clash-meta snippet. The generated config uses TLS + WebSocket with 0-RTT early data (`/?ed=2048`).

**Preferred IPs (优选 IP):** to pick a better entry datacenter, replace only the server address in the client config with any Cloudflare edge IP that tests well from your ISP, and keep the WebSocket `Host` header / TLS SNI set to your worker domain. Verify the entry colo via the `colo` field at `GET /`.

## Region pinning (Placement)

By default a Worker runs at the datacenter closest to the visitor. To pin it to a region instead, set a Placement hint (`Workers & Pages → Settings → Runtime → Placement`, e.g. region `azure:westus`). Verify with the `cf-placement` response header (`remote-SJC` = running remotely in San Jose).

Note: `[placement] region` in `wrangler.toml` requires Wrangler 4+; this repo pins Wrangler 3, so placement is managed in the dashboard, and a re-deploy may reset it — re-check after deploying.

## Local development

`config/config-client-with-*-local.json` are example Xray client configs for pairing with `wrangler dev` (SOCKS :4080 / HTTP :4081 inbounds). `test/` contains the original author's standalone experiment scripts and is kept as-is.

## Acknowledgements

- [zizifn/edgetunnel](https://github.com/zizifn/edgetunnel) — the original project this fork builds on.

## License

GPL-2.0 — see [LICENSE](LICENSE). Forks must keep the source open.
