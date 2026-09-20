# 在 Lenspace fork 上发布和测试

## 已保存的上游 PR 基线

两个仓库都保留了 `pr/report-only` 分支，只包含只上报模式的功能改动：

| 仓库 | 功能提交 |
|---|---|
| `Lenspace/agent` | `2b81d62` — agent 只上报模式及测试 |
| `Lenspace/monitor` | `0d65d13` — 安装时选择模式及文档 |

`monitor` 的 `main` 另外包含 fork 部署配置：hub 从 `Lenspace/agent` 下载 agent，
hub 安装器从 `Lenspace/monitor` 下载 hub。这些配置不属于上游 PR。
agent 安装脚本 `install.sh` 仍从当前 hub 下载二进制，下载来源由 hub 内的 `AGENT_REPO` 决定。
默认主题继续使用原作者的发布包和原有 SHA-256 校验，无需另外 fork 主题。

## 1. 启用 GitHub Actions

分别打开：

- <https://github.com/Lenspace/agent/actions>
- <https://github.com/Lenspace/monitor/actions>

如果 GitHub 提示 fork 的工作流已停用，点击启用。
现有工作流已声明发布所需的权限，公开仓库的正常构建不需要手动创建 PAT 或添加 secret。
如果仓库或组织限制第三方 Actions，需允许工作流里已有的 actions 和 dtolnay/rust-toolchain。

- 推送 `main`：运行 `ci`，检查格式、代码和测试。
- 推送 `v*` 标签：运行 `release`，编译 Linux x86_64 / aarch64 静态二进制并创建 GitHub Release。
- 在 Actions 对 `main` 手动运行 `release`：只编译并上传 Artifacts，不创建 Release，安装器无法从 Artifacts 下载。
- hub 发布还会推送 `ghcr.io/lenspace/monitor` 镜像；没有 Docker Hub secrets 时会自动跳过 Docker Hub。
  本文使用二进制安装，不依赖容器镜像。

## 2. 推送代码，等 CI 通过

以下 Git 命令在同时包含 `agent/` 和 `monitor/` 子目录的工作区外层执行：

```bash
git -C agent push origin main pr/report-only
git -C monitor push origin main pr/report-only
```

分别在 Actions 确认 `ci` 通过，再推送发布标签。`release` 工作流本身不等待 `ci`。
当前 remote 使用 SSH，推送需要本机的 GitHub SSH key 有这两个仓库的写权限。

## 3. 先发布 agent，再发布 hub

下面以本 fork 的 `v1.1.1` 标签为例；后续使用新的、未发布过的标签，不要覆盖旧标签。
标签控制发布和下载版本，程序显示的版本仍取自 `Cargo.toml`；本次未修改其中的 `1.1.0`。

```bash
git -C agent tag -a v1.1.1 main -m "Test report-only agent on Lenspace fork"
git -C agent push origin v1.1.1
```

等待 agent 的 `release` 成功，在 <https://github.com/Lenspace/agent/releases> 确认存在：

- `monitor-agent-x86_64-unknown-linux-musl`
- `monitor-agent-aarch64-unknown-linux-musl`
- `sha256sums.txt`

然后发布 hub：

```bash
git -C monitor tag -a v1.1.1 main -m "Test report-only installer on Lenspace fork"
git -C monitor push origin v1.1.1
```

等待 <https://github.com/Lenspace/monitor/releases> 出现两个 `monitor-hub-…-unknown-linux-musl`
文件和 `sha256sums.txt`。两边安装器都使用 `releases/latest/download`，发布应是普通 Release，
不要设为 Draft 或 Pre-release，并确认显示为 Latest。只推送代码或手动编译不足以安装。

## 4. 在测试服务器安装 hub

在一台测试用 Linux systemd 主机上执行。安装器使用固定的 `monitor-hub` 服务和 `/opt/monitor` 目录，
如果该机器已有安装，会执行升级并复用原数据库。

```bash
curl -fsSL https://raw.githubusercontent.com/Lenspace/monitor/main/install-hub.sh -o install-hub.sh
sudo sh install-hub.sh
```

选择“安装 / 升级”。hub 默认只监听 `127.0.0.1:28080`，需要把你自己的 HTTPS 域名反向代理到这个地址。
例如已安装 Caddy，且域名已解析到测试服务器，可使用：

```caddyfile
hub.example.com {
    reverse_proxy 127.0.0.1:28080
}
```

把 `hub.example.com` 换成实际域名，并应用到你已有的 Caddy 配置。
访问 `https://你的域名/admin`，使用安装器打印的初始密码登录；也可在本机通过
`sudo journalctl -u monitor-hub` 查看首次启动日志。当前面板要求 HTTPS 域名才能添加和安装节点。

## 5. 安装 agent，验证两种模式

在面板添加节点，复制生成的安装命令，在 agent 测试主机的 root shell 执行。
安装器会询问模式；也可以在面板命令末尾追加：

```text
--report-only
```

开启只上报后，检查：

```bash
sudo grep '^MONITOR_REPORT_ONLY=' /opt/monitor/agent.env
sudo systemctl cat monitor-agent
sudo journalctl -u monitor-agent -n 50 --no-pager
```

应看到 `MONITOR_REPORT_ONLY=true`、启动参数 `--report-only`，以及日志
`report-only mode: ignoring all hub tasks`。面板中的节点应在线且指标持续刷新。

在面板给该节点分配一个你控制的 TCP 探测目标，目标端用抓包或连接记录检查：
只上报模式不应产生探测连接或新的探测结果。改用同一节点的安装命令并追加 `--allow-probes` 重装，
应恢复探测；再用 `--report-only` 重装，应停止新增探测。历史探测数据不会因为切换模式自动删除。
不要同时传入两个模式参数。

不指定参数的重装默认保留已有模式；首次安装默认常规探测。无终端时也遵循此规则。
只上报模式阻止运行时下发的任务，安装和升级时仍需信任脚本、二进制及提供它们的 hub。

## 6. 向原作者提 PR

测试完成后，分别在 GitHub 创建 PR：

- base `monitor-probe/agent:main` ← head `Lenspace/agent:pr/report-only`
- base `monitor-probe/monitor:main` ← head `Lenspace/monitor:pr/report-only`

不要用包含个人下载地址配置的 `monitor:main` 提功能 PR。
如果测试中还要修功能问题，把功能修复提交也放到对应的 `pr/report-only` 分支，再合并回 `main` 继续测试。
上游需要先发布支持新参数的 agent，再发布安装脚本。
