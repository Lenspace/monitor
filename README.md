# monitor

本 fork 的编译、发布、安装测试与上游 PR 步骤见 [FORK_TESTING.md](FORK_TESTING.md)。

## 特性

- 实时监控：秒级实时数据展示
- 轻量高效：Rust 语言构建，低资源占用，极简高效
- 自托管：完全掌控数据隐私，部署简单
- 通知：节点掉线、流量、到期与登录，推送到 Telegram 或自定义 Webhook

## 组成

| 仓库 | 说明 |
|---|---|
| [monitor](https://github.com/monitor-probe/monitor) | hub：后台、API、公开页宿主 |
| [agent](https://github.com/monitor-probe/agent) | Linux agent |
| [monitor-theme-default](https://github.com/monitor-probe/monitor-theme-default) | 内置默认主题 |

```
agent (Linux)  ──WebSocket / JSON-RPC 2.0──▶  hub (axum + SQLite)  ──▶  后台 + 状态页
```

## Agent 安装模式

执行面板提供的安装命令时，脚本会询问模式：

- **只上报**：上报本机指标，忽略 hub 下发的所有任务，不进行 TCP 探测。
- **常规探测**：上报指标并执行 hub 下发的 TCP 探测任务，保持原有行为。

命令末尾加 `--report-only` 或 `--allow-probes` 可直接指定，适用于单节点和 `--register` 批量安装。
首次安装默认常规探测；重装默认保留 `/opt/monitor/agent.env` 中的选择。没有终端时直接使用该默认值。
只上报模式由 agent 本机强制执行，主控无法通过下发任务解除；指标上报和连接心跳正常工作。

需先发布支持 `--report-only` 的 agent，再发布安装脚本。旧 agent 收到该参数会拒绝启动，避免误以为
已启用保护。此模式不是“绝对安全”的保证；安装脚本和二进制仍由 hub 提供，安装及升级时必须信任来源。
