# RCA：`hermes update` 后 SSL CA 证书 bundle 损坏

**状态：** 已由 `fix(ssl): surface broken CA bundles before provider calls` 解决
**严重级别：** P2 —— 在用户修复依赖或 CA 配置之前，会使 agent 陷入不透明的 provider/客户端失败状态。

## 摘要

部分的 `hermes update`、被中断的 venv 修复，或陈旧的 CA-bundle 环境变量，可能使 Python TLS 配置指向一个缺失、空或无法加载的 CA bundle。随后，首次出站 HTTPS 客户端创建或请求可能会以原始的 `FileNotFoundError: [Errno 2] No such file or directory` 或一个不会指明损坏 CA 路径的低层 SSL 错误而失败。

## 根本原因

Hermes 使用基于 OpenAI/httpx 和 requests 的客户端进行 provider 调用、模型元数据获取、gateway 投递以及 web 工具操作。这些客户端从以下来源继承 CA bundle 设置：

- `HERMES_CA_BUNDLE`
- `SSL_CERT_FILE`
- `REQUESTS_CA_BUNDLE`
- `CURL_CA_BUNDLE`
- 内置 `certifi` 包的 `cacert.pem`

当 venv 被部分刷新时，或当上述某个环境变量指向一个已不存在的文件时，provider 客户端构造过程可能在 Hermes 拥有足够上下文产出有用错误信息之前就已失败。

## 修复方案

`agent/ssl_guard.py` 会在 `agent/agent_init.py` 中创建 OpenAI 兼容 provider 客户端之前校验 CA bundle 配置。它会：

1. 检查显式的 CA bundle 环境变量，并报告确切的损坏变量/路径，
2. 验证 `certifi` 可被导入，
3. 验证 `certifi.where()` 指向一个大小合理的现存文件，
4. 从每个被检查的 bundle 构建一个 `ssl.SSLContext`，
5. 在 httpx/OpenAI 抛出原始低层错误之前，抛出一个带修复提示的类型化 `SSLConfigurationError`。

`hermes_cli doctor` 在 `SSL / CA Certificates` 项下提供了相同的检查，因此用户无需启动模型会话即可诊断问题。

## 恢复方式

当该防护在 agent 初始化时触发，用户会看到类似如下的消息：

```text
Failed to initialize OpenAI client: SSL_CERT_FILE points to a missing CA bundle: C:\path\to\missing\cacert.pem
Repair: python -m pip install --force-reinstall certifi openai httpx
If you configured a custom corporate CA bundle, fix or unset the broken CA bundle environment variable.
```

对于普通的 Hermes venv 损坏情形，重新安装受影响的客户端依赖：

```bash
python -m pip install --force-reinstall certifi openai httpx
```

对于自定义/企业 CA 配置，请修复环境变量使其指向一个真实的 PEM bundle；如果希望 Hermes 使用内置的 `certifi` 证书库，则取消设置该变量。

## 环境变量逃生开关

设置 `HERMES_SKIP_SSL_GUARD=1` 可跳过该预检。这仅适用于沙箱化或受托管信任的环境——在这些环境中 Python CA 路径看起来异常，但下游客户端已知可正常工作。
