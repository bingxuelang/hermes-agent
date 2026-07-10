# 多网关部署

Hermes 支持多个 gateway 进程并发运行——每个 profile（default、writer、admin、coder、researcher）对应一个。每个 gateway 独立建立与平台 API 的连接，并为其 profile 的订阅者投递消息。

## 单 dispatcher 模式

仅有一个 gateway 拥有 kanban dispatcher。拥有该职责的 gateway 保持 `kanban.dispatch_in_gateway: true`（默认值）；其余所有 gateway 将其设为 `false`。

**重要性说明：** 一个设置了 `dispatch_in_gateway: true` 的 gateway 会为 dispatcher 和 notifier watcher 分别打开针对每个 board 的 SQLite 连接。多个 gateway 并发执行此操作会使每个 `kanban.db` 上的打开文件描述符成倍增加，并加剧 WAL `-shm` 读取者争用。将两条路径都由同一标志控制，可确保仅有一个进程触碰 kanban 数据库。

## 配置

在拥有 dispatch 职责的 gateway 上（通常是 `default` profile），无需改动。在其余每个 profile 的 gateway 上，向 `~/.hermes/config.yaml` 添加：

```yaml
kanban:
  dispatch_in_gateway: false
```

或设置环境变量：`HERMES_KANBAN_DISPATCH_IN_GATEWAY=false`

## 各 gateway 的职责

| Gateway 角色 | dispatch_in_gateway | 打开每个 board 的 DB？ | 运行 dispatcher + notifier？ |
|---|---|---|---|
| default（dispatch 拥有者） | true（默认） | 是 | 是 |
| writer、admin、coder 等 | false | 否 | 否 |

非 dispatch 的 gateway 仍会为其自身的平台适配器
（Telegram、Discord 等）投递消息——它们只是不会轮询 kanban board。
