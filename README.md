# codex-proxy

macOS 版 [Codex](https://github.com/openai/codex) 通过 Clash 代理联网的启动脚本。解决不开启 TUN 模式时，HTTP 流量走代理但 WebSocket 不走代理，导致移动端 ChatGPT 无法远程连接电脑 Codex App 的问题。不影响其他 App 的代理设置。

## 背景

macOS 下 Codex App + Clash 环境，如果不开启 TUN 模式，仅 HTTP 流量会走系统代理，WebSocket 连接不会经过代理。Codex App 远程连接功能依赖 WebSocket，因此会出现移动端无法连接远程电脑的情况，Codex App 界面显示 `reconnecting 1/5` 到 `5/5` 后连接失败。

开启 TUN 模式虽然可以解决，但会影响整机所有网络流量。此脚本通过环境变量注入 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`，使 Codex 进程的 HTTP 和 WebSocket 流量全部经过 Clash 代理，无需开启 TUN 模式。

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
