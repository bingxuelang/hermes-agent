# Docker 部署的网络出口隔离

在 Docker 中运行 Hermes 时，默认的 `network_mode: host` 会赋予
agent 进程不受限制的出站网络访问能力。本指南介绍如何对流量进行分段，使 agent 核心只能
访问其所需的服务，同时阻止任意的出站连接。

这主要是为了防御提示注入（prompt injection）攻击——此类攻击会尝试通过工具生成的 shell
命令中的 `curl`、`wget` 或原始 HTTP 请求来外泄数据。

## 威胁模型

Hermes 的 [SECURITY.md](../../SECURITY.md) §2 定义了信任模型。终端
后端是主要的执行边界。然而，当以
`network_mode: host` 运行时，agent 执行的任何命令都可以访问网络上的任意端点，
包括外部端点。

网络出口隔离增加了第二层防护：即使恶意命令
在容器内执行，它也无法触达显式白名单集合之外的端点。

## 架构

```
┌─────────────────────────────────────────────┐
│  Docker Network: internal (no internet)     │
│                                             │
│   ┌──────────────┐   ┌──────────────────┐   │
│   │ hermes-agent │   │ hermes-dashboard │   │
│   └──────┬───────┘   └────────┬─────────┘   │
│          │                    │              │
│          ▼                    │              │
│   ┌──────────────┐            │              │
│   │ hermes-gtw   │◄───────────┘              │
│   └──────┬───────┘                           │
│          │                                   │
└──────────┼───────────────────────────────────┘
           │
┌──────────┼───────────────────────────────────┐
│  Docker Network: egress (internet-capable)   │
│          │                                   │
│          ▼                                   │
│   ┌─────────────────┐                        │
│   │ egress-proxy     │──► allowlisted hosts  │
│   │ (squid / envoy)  │                       │
│   └─────────────────┘                        │
└──────────────────────────────────────────────┘
```

两个 Docker 网络：

- **`internal`** —— 无默认路由，无互联网访问。agent、dashboard
  和 gateway 在此运行。
- **`egress`** —— 具有互联网访问能力。仅需要访问外部
  API 的服务才会接入此网络。

gateway 服务采用双宿主（dual-homed）方式（同时接入两个网络），以便
接收来自 Telegram/Slack 等平台的入站消息，并将其转发给
内部网络上的 agent。

## Compose 配置

使用 `docker-compose.override.yml` 覆盖默认的
`docker-compose.yml`：

```yaml
# docker-compose.override.yml
# Network egress isolation for production deployments.
#
# Usage:
#   HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d
#
# This overrides network_mode: host with isolated Docker networks.

networks:
  internal:
    driver: bridge
    internal: true          # no default route, no internet
  egress:
    driver: bridge

services:
  gateway:
    network_mode: ""        # clear the host-mode default
    networks:
      - internal
      - egress              # needs outbound for Telegram, LLM APIs
    ports:
      - "127.0.0.1:9119:9119"   # dashboard proxy, localhost only

  dashboard:
    network_mode: ""
    networks:
      - internal            # internal only, no egress needed
```

### 使用出口代理（推荐）

为了实现更严格的控制，可通过带有显式白名单的 HTTP 代理
路由所有出站流量：

```yaml
# docker-compose.override.yml (with egress proxy)

networks:
  internal:
    driver: bridge
    internal: true
  egress:
    driver: bridge

services:
  gateway:
    network_mode: ""
    networks:
      - internal
      - egress
    environment:
      - HTTP_PROXY=http://egress-proxy:3128
      - HTTPS_PROXY=http://egress-proxy:3128
      - NO_PROXY=hermes,hermes-dashboard,localhost

  dashboard:
    network_mode: ""
    networks:
      - internal

  egress-proxy:
    image: ubuntu/squid:6.10-24.04_edge
    networks:
      - egress
    volumes:
      - ./config/squid-allowlist.conf:/etc/squid/conf.d/allowlist.conf:ro
    restart: unless-stopped
```

示例 `config/squid-allowlist.conf`：

```
# Only allow HTTPS CONNECT to these hosts
acl allowed_hosts dstdomain api.openai.com
acl allowed_hosts dstdomain api.anthropic.com
acl allowed_hosts dstdomain openrouter.ai
acl allowed_hosts dstdomain generativelanguage.googleapis.com
acl allowed_hosts dstdomain api.telegram.org
acl allowed_hosts dstdomain api.github.com
acl allowed_hosts dstdomain discord.com

http_access allow CONNECT allowed_hosts
http_access deny all
```

请根据你的 LLM 提供商和消息平台调整白名单。

## 验证配置

启动整个栈之后，验证隔离效果：

```bash
# From the agent container: this should FAIL (no egress)
docker compose exec gateway \
  curl -sf --max-time 5 https://example.com && echo "FAIL: egress not blocked" || echo "OK: egress blocked"

# From the agent container: this should SUCCEED (internal network)
docker compose exec gateway \
  curl -sf --max-time 5 http://hermes-dashboard:9119/health && echo "OK: internal reachable" || echo "FAIL"

# If using egress proxy: this should SUCCEED (allowlisted)
docker compose exec gateway \
  curl -sf --max-time 5 --proxy http://egress-proxy:3128 https://api.openai.com/v1/models && echo "OK" || echo "FAIL"
```

## 局限性

- **DNS 解析：** 除非你同时运行一个阻止外部查询的本地 DNS 解析器，
  否则 `internal` 网络仍然可以解析外部 DNS
  名称。对于大多数威胁模型而言这是可接受的，因为仅 DNS 解析本身不会
  外泄有意义的数据。

- **不能替代沙箱后端：** 本指南隔离的是 agent
  *容器*的网络。如果你使用默认的本地终端后端，工具
  命令会在同一容器内执行。为了获得更强的隔离性，请将
  网络分段与沙箱化的终端后端（Docker、Modal、
  Daytona）结合使用。

- **平台适配器需要出口访问：** gateway 服务需要出站访问
  才能触达消息平台 API。如果新增平台适配器，请将其
  API 端点加入代理白名单。

## 相关文档

- [SECURITY.md](../../SECURITY.md) —— Hermes 信任模型与漏洞报告
- [Terminal backends](../../README.md) —— 沙箱化执行目标
- [docker-compose.yml](../../docker-compose.yml) —— 默认 compose 配置
