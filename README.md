# codex-proxy

macOS 版 [Codex](https://github.com/openai/codex) 通过 Clash 代理联网的启动脚本。解决不开启 TUN 模式时，HTTP 流量走代理但 WebSocket 不走代理导致的两个问题。仅影响 Codex 进程本身，不影响其他 GUI App 的代理设置。

## 背景

macOS 下 Codex App + Clash 环境，如果不开启 TUN 模式，仅 HTTP 流量会走系统代理，WebSocket 连接不会经过代理。这会引发两个独立的问题：

**问题 A：移动端 ChatGPT 无法远程连接电脑 Codex App**

Codex 的[远程连接功能](https://developers.openai.com/codex/remote-connections)依赖 WebSocket 长连接，手机端 ChatGPT App 通过 WebSocket 与电脑端 Codex 通信。WebSocket 不走代理时，手机端无法发现或连接到电脑。

**问题 B：Codex App 自身反复重连（Reconnecting 1/5 ∼ 5/5）**

Codex App 与 OpenAI 后端之间同样依赖 WebSocket 维持长连接。WebSocket 不走代理时，App 连接后端失败，界面显示 `reconnecting 1/5` 到 `5/5`，重试 5 次后放弃。

开启 TUN 模式可以同时解决这两个问题，但 TUN 是虚拟网卡级别的全局代理，会影响整机所有 App 的网络流量。此脚本通过环境变量注入 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`，只让 Codex 这一个进程的 HTTP 和 WebSocket 流量走 Clash 代理，其他 GUI App 不受任何影响。

## 前置条件

- macOS
- [Clash](https://github.com/Dreamacro/clash) 或兼容客户端（Clash Verge、ClashX 等）
- 代理端口默认 `7890`（Clash 默认端口）

## 安装

```bash
mkdir -p ~/bin

# 将 codex-proxy 脚本复制到 ~/bin/
cp codex-proxy ~/bin/codex-proxy
chmod +x ~/bin/codex-proxy

# 确保 ~/bin 在 PATH 中
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

## 使用

```bash
codex-proxy start     # 启动 Codex（带代理）
codex-proxy stop      # 退出 Codex
codex-proxy restart   # 重启 Codex
codex-proxy status    # 查看运行状态
codex-proxy log       # 实时查看日志
```

## 自定义配置

通过环境变量覆盖默认值：

```bash
# 自定义 Codex App 路径
export CODEX_APP="/Applications/Codex.app"

# 自定义 HTTP 代理地址
export CODEX_HTTP_PROXY="http://127.0.0.1:7890"

# 自定义 SOCKS5 代理地址
export CODEX_ALL_PROXY="socks5://127.0.0.1:7890"

# 自定义不走代理的地址列表
export CODEX_NO_PROXY="localhost,127.0.0.1,::1"
```

可将以上配置写入 `~/.zshrc` 持久化。

## 原理

Codex App 启动时，通过 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 等环境变量注入代理配置，使 Codex 进程的所有网络请求经过 Clash 代理转发。脚本在启动前会先退出已有的 Codex 进程，确保不带代理的残留进程不存在。

## 日志

日志文件位于 `~/Library/Logs/codex-proxy.log`。

## License

MIT
