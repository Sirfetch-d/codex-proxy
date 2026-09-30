# codex-proxy

macOS 版 ChatGPT（原 Codex）桌面 App 走本地代理（Clash / Clash Verge / v2rayN / Surge …）的启动脚本。
**不需要 TUN 模式，也不影响其它 App**：只有 ChatGPT 这一个 App 的流量走代理。

```bash
codex-proxy start     # 启动（自动探测系统代理，注入 Chromium --proxy-server + 环境变量）
codex-proxy persist   # 让后台 app-server 守护进程也走代理（手机端显示离线时必做）
codex-proxy status    # 看「到底有没有真的走代理」
```

---

## 新版 App 变了什么（重要）

2026 年年中起，ChatGPT 桌面版从原生 App 换成了 **Electron（Chromium）内核**
（`Contents/Resources/app.asar` + `Codex Framework.framework`，本机版本 `26.924.22138`）。
这次重写让「只注入 `HTTP_PROXY` 环境变量」的老做法**彻底失效**，原因是有两条完全独立的网络路径：

| 网络路径 | 谁在用 | 认什么代理 |
| --- | --- | --- |
| **Chromium 网络栈** | 界面与网页请求、`electron.net.fetch`、Statsig、`ws.chatgpt.com` 等 WebSocket | macOS 系统代理，或启动参数 `--proxy-server`。**在 macOS 上不读取 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 环境变量**（只有 Linux 版 Chromium 才读） |
| **Rust `codex`** | `codex app-server`、模型请求、远程连接的**被控端** WebSocket、MCP | `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 等环境变量，以及 `$CODEX_HOME/.env` |

本机实测（macOS 27 + ChatGPT `26.924.22138`，代理端用日志代理观察真实连接）：

| 启动方式 | 代理端收到的连接 |
| --- | --- |
| 只注入 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`（老脚本做法） | **0** |
| 加 `--args --proxy-server=http://127.0.0.1:7890` | `chatgpt.com:443`、`ws.chatgpt.com:443`、`developers.openai.com:443` … 全部出现 |

因此本脚本现在**同时**处理两层：

1. **Chromium 层**：`open -a ChatGPT.app --args --proxy-server=… --proxy-bypass-list=…`
   → 界面、所有 REST 请求、以及 `ws.chatgpt.com` 这类 WebSocket 一起走代理。
2. **Rust 层**：`--env HTTP_PROXY/HTTPS_PROXY/ALL_PROXY/NO_PROXY`（大小写各一份），
   另外提供 `codex-proxy persist` 把代理写进 `~/.codex/.env`
   → 后台 `app-server` 守护进程、终端里的 `codex` 在任何启动方式下都走代理。

> 后台守护进程的坑：`codex app-server`（模型请求、远程连接被控端）由常驻 supervisor
> `codex app-server daemon pid-update-loop` 派生，**ppid=1 脱离于 App**，重启 App 不会刷新它的环境变量。
> 结果就是界面正常、模型请求却一直超时（`app-server.stderr.log` 里反复出现
> `failed to refresh available models: request timed out`）、手机端显示「Mac 离线」。
> 本脚本会在启动前清理这种「没有代理环境变量」的旧守护进程，并可用 `persist` 一劳永逸。

---

## 对应修复的 issue

| Issue | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| [#3](../../issues/3) | 启动白屏 / 卡 logo，日志里 `sa_server_request_failed`、Statsig bootstrap 超时 | Chromium 层没走代理（环境变量对它无效） | `--proxy-server` |
| [#4](../../issues/4) | macOS 27 卡在 logo 页，开 TUN 秒进 | 同上：请求直连超时 | `--proxy-server` |
| [#2](../../issues/2) | 启动很慢、日志大量 timeout、`net::ERR_CONNECTION_TIMED_OUT` | 上半段：Chromium 层没走代理；下半段：`app-server` 守护进程没走代理 | `--proxy-server` + 环境变量 + 清理旧守护进程 |
| [#5](../../issues/5) | 「搞的都无法正常用了」（白屏 / 转圈） | 同 #3 / #4 | 同上 |
| [#1](../../issues/1) | 手机端一直显示「Mac 离线」，无法远程连接 | 远程连接的**被控端**长连接在 Rust `codex` 里，Chromium 的 `--proxy-server` 管不到 | `codex-proxy persist`（写 `~/.codex/.env`）；若仍不行见「排错」 |

顺带修复的老毛病：端口写死 `7890`（Clash Verge 用 7897、v2rayN 用 10809 的用户会直接失败）。
现在**默认自动读取 macOS 系统代理**（`scutil --proxy`），换客户端、换端口都不用改脚本。

---

## 前置条件

- macOS 13 或更高（`open --env` / `--args` 需要较新系统；更老的系统会自动降级为直接启动）
- Clash / Clash Verge / v2rayN 等本地代理
- 系统设置里已开启 HTTP/HTTPS 代理（推荐），或者用 `CODEX_PROXY` 指定

## 安装

```bash
mkdir -p ~/bin
cp codex-proxy ~/bin/codex-proxy
chmod +x ~/bin/codex-proxy
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

## 使用

```bash
codex-proxy start        # 启动（若已在运行且代理正确，则不会重启）
codex-proxy restart      # 强制重启
codex-proxy status       # 运行状态 + 是否真的注入了代理
codex-proxy stop         # 完全退出（含后台 app-server 守护进程）
codex-proxy log          # 实时看 App 输出（tail -f）
codex-proxy doctor       # 一键自检：连通性、端口、内核、守护进程、持久化配置
codex-proxy persist      # 写 ~/.codex/.env，让 Rust 侧持久走代理（等价命令：codex-proxy unpersist 撤销）
codex-proxy persist off  # 撤销上面的写入
codex-proxy persist show # 查看 ~/.codex/.env
```

`stop` 默认会连后台 `app-server` 守护进程一起停掉（会中断正在跑的后台任务）；
想保留后台任务用 `codex-proxy stop --keep-daemon`。

## 自定义配置

```bash
# 指定代理（不设则自动读取 macOS 系统代理，再退回 http://127.0.0.1:7890）
export CODEX_PROXY="http://127.0.0.1:7897"          # 旧名字 CODEX_HTTP_PROXY 仍兼容
export CODEX_ALL_PROXY="socks5://127.0.0.1:7897"    # 默认与 CODEX_PROXY 相同
export CODEX_NO_PROXY="localhost,127.0.0.1,::1,*.local"

# 指定 App 路径（默认自动查找 /Applications/ChatGPT.app，也兼容 Codex.app）
export CHATGPT_APP="/Applications/ChatGPT.app"

# codex 配置目录（默认 ~/.codex），persist 写在其下的 .env
export CODEX_HOME="$HOME/.codex"

# 日志位置
export CODEX_PROXY_LOG="$HOME/Library/Logs/codex-proxy.log"
```

写进 `~/.zshrc` 即可持久化。

## 命令输出示例

```
$ codex-proxy status
ChatGPT 状态
  App        : /Applications/ChatGPT.app
  版本       : 26.924.22138 (Electron)
  期望代理   : http://127.0.0.1:7890  (macOS 系统代理)

  App 进程   : 运行中（PID 32592，本脚本记录 32592）
✓   Chromium 代理参数：--proxy-server=http://127.0.0.1:7890（与配置一致）
✓   代理环境变量：HTTPS_PROXY=http://127.0.0.1:7890（子进程生效）

  后台受管守护进程（app-server daemon）：未运行
```

被双击启动（没有注入 `--proxy-server`）时会明确报出来：

```
✗   未注入 --proxy-server —— 新版 Electron App 忽略环境变量，它的流量没有走代理！
     修复：codex-proxy restart
```

## 原理

1. 退出已有 ChatGPT 进程（AppleScript + `pkill`，并清理 bundle 内残留的
   Chromium 辅助进程 / `cua_node` / 内置 codex）。
2. 如果后台 `app-server` 守护进程没有代理环境变量，先把它停掉（官方
   `codex app-server daemon stop`，兜底 kill 掉脱离的 supervisor），
   让新进程从带代理环境的 App 继承环境变量。
3. `open -a ChatGPT.app --stdout/--stderr 日志 --env 代理变量 --args --proxy-server=… --proxy-bypass-list=…`
   - `--proxy-bypass-list` 默认是 `localhost;127.0.0.1;::1;*.local;<local>`（由 `CODEX_NO_PROXY` 转换而来）；
     **绝不要加 `<-loopback>`**：代理本身就在 `127.0.0.1` 上，取消回环绕行会形成自环。
   - `--stdout/--stderr` 必须写在 `--args` 之前，否则会被当成 App 参数。
   - 启动前会确认旧进程确实退干净（`open` 不会给已在运行的实例重新传参），
     并断言新进程的 `--proxy-server` 真的生效，失败时返回非 0。
4. 校验真实进程的命令行参数与环境变量，把主进程 PID 写进
   `~/Library/Application Support/codex-proxy/codex.pid`，启动参数写进同目录的 `session.env`。
   日志超过 5 MB 时自动轮转一份到 `codex-proxy.log.1`。

Chromium 的 `--proxy-server` 对 `https://` 与 `wss://` 一视同仁（都走 CONNECT 隧道），
所以远程连接、`ws.chatgpt.com` 这类 WebSocket 都会走代理。

## 排错

| 症状 | 处理 |
| --- | --- |
| `doctor` 说「没有 --proxy-server」 | App 是被双击 / 旧脚本启动的，执行 `codex-proxy restart` |
| 代理连不上（`HTTP 000`） | 代理软件没开或端口变了；用 `CODEX_PROXY=http://127.0.0.1:端口` 指定，或确认系统代理是否开启 |
| 界面正常但模型请求超时 / 手机端「Mac 离线」 | 后台守护进程没走代理：`codex-proxy persist`，然后 `codex-proxy restart` |
| 手机端仍然离线 | 看 `~/.codex/app-server-daemon/app-server.stderr.log`；已知非代理原因：`state_5.sqlite` 损坏导致 app-server 起不来，需要在 App 内重置远程连接（参考 [#1](../../issues/1) 的说明） |
| App 白屏 / 卡 logo，但 `doctor` 全绿 | 可能是 App 自身渲染进程卡死（与代理无关），`codex-proxy restart` 通常能恢复 |
| 想彻底撤销 | `codex-proxy stop`（退出 App）+ `codex-proxy persist off`（撤销持久化） |

其他排查命令：

```bash
# App 主进程有没有 --proxy-server
ps -o command= -p "$(pgrep -f '^/Applications/ChatGPT.app/Contents/MacOS/ChatGPT')"

# 谁在连代理（Chromium 的网络进程是 "Codex (Service) ... network.mojom.NetworkService"）
lsof -nP -iTCP | grep 127.0.0.1:7890

# 官方自带诊断（也能看出 Rust 侧认到的代理）
codex doctor --json | python3 -c "import json,sys;print(json.load(sys.stdin)['checks']['network.env']['details'])"
```

## 已知限制

- Chromium 层的 `--proxy-server` 只在**启动时**生效：`open` 不会给已在运行的实例重新传参，
  所以脚本每次都会先退出 App；`codex-proxy start` 对已正确配置的实例不会重复重启。
- Electron 主进程里用 Node `ws` 发起的少数连接既不读环境变量，也不吃 `--proxy-server`
  （官方没有任何桌面端出站代理设置项，feature request 仍处于 open 状态）。
  远程连接真正依赖代理的是 **Rust 侧**，也就是 `persist` 覆盖的那部分。
- `--proxy-server` 作用于整个 App 实例的 Chromium 网络栈，无法只代理其中一部分域名。
- 本脚本不修改系统代理、不启用 TUN，也不影响其它 App；`persist` 只写
  `$CODEX_HOME/.env`（默认 `~/.codex/.env`）里带标记的一小段，权限保持与写入前一致
  （新文件为 `600`），**不会动你在这个文件里的其它内容**，也可以用 `persist off` 撤销
  （如果撤销后文件只剩空白，会直接删除该文件）。如果发现标记不完整（只写了 `# >>> codex-proxy >>>`
  没有结束标记），脚本会拒绝改动并提示你手工修正，避免误删你的配置。

## License

MIT
