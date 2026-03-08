# 在 Claude Code 中通过 LiteLLM 使用 GitHub Copilot Provider 的 Claude 模型

> 文档日期：2026-03-09  
> 这份文档基于今天可用的行为整理。GitHub Copilot、LiteLLM、Claude Code 的接口和鉴权流程后续都可能变化，一些步骤可能会需要调整。

## 这份文档解决什么问题

目标是把 LiteLLM 的 GitHub Copilot provider 暴露成一个本地 OpenAI 兼容服务，再让 Claude Code 通过自定义 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 接到这个本地服务上。

这样做之后，Claude Code 可以把请求发到本机 LiteLLM，再由 LiteLLM 转发到 GitHub Copilot 提供的 Claude 系列模型。

这份文档的目标读者是 Claude Code 的开发人员使用者，主要提供以下内容：

- 要准备什么
- 具体怎么配置
- 第一次怎么登录
- 平时怎么稳定使用
- 哪些限制要提前知道

## 前提

- 已安装 LiteLLM
- 已安装 Claude Code
- 本机可访问 GitHub Copilot 相关服务
- 你有可用的 GitHub Copilot 账户

## 限制和风险

- LiteLLM 提供的 GitHub Copilot provider 中的 Claude 模型，不等同于 Anthropic 原生 Claude API。
- Claude Code 发出的部分参数可能不被 LiteLLM 的 GitHub Copilot provider 支持。比如 `thinking`、`context_management` 这类 Claude 特有参数，通常需要依赖 `drop_params: true` 丢弃，否则会报错。
- 上下文窗口基本跟 GitHub Copilot 侧保持一致，量级约为 200k，但实际可用容量仍取决于具体模型和服务端行为。
- GitHub Copilot 当前更偏向按调用次数而不是按 token 的方式计量。通过 Claude Code + LiteLLM 转发使用时，调用次数可能比在 GitHub Copilot 原生 Agent 中更高。
- 个人自用被GitHub Copilot封禁的风险相对低一些，参考opencode使用github copilot模型的方式[链接](https://github.blog/changelog/2026-01-16-github-copilot-now-supports-opencode/)，但多人共享同一个 LiteLLM 中转服务会有比较高的封号风险，不建议团队共用。

## 整体流程

1. 配置 LiteLLM，让它把某个本地模型名映射到 GitHub Copilot 的 Claude 模型。
2. 启动 LiteLLM 本地服务。
3. 用一次测试请求触发 GitHub OAuth device flow 登录。
4. 把 Claude Code 指向本机 LiteLLM。
5. 后续通过 Claude Code 正常使用该模型。

## LiteLLM 配置

下面给出一个最小可用的 `config.yaml` 示例：

```yaml
model_list:
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: github_copilot/claude-sonnet-4.6
      drop_params: true
      extra_headers:
        Editor-Version: "vscode/1.110.1"
        Editor-Plugin-Version: "copilot-chat/0.38.2"
        Copilot-Integration-Id: "vscode-chat"
        User-Agent: "GitHubCopilotChat/0.38.2"

general_settings:
  master_key: sk-your-local-litellm-key
```

可选模型通常包括：

- `github_copilot/claude-sonnet-4.6`
- `github_copilot/claude-sonnet-4.5`
- `github_copilot/claude-opus-4.6`
- `github_copilot/claude-opus-4.5`
- `github_copilot/claude-haiku-4.5`

说明：

- `model_name` 是你暴露给 Claude Code 使用的本地名字。
- `drop_params: true` 很重要，用来丢弃 Claude Code 发送但后端不认识的参数。
- `master_key` 是你本地 LiteLLM 服务的访问密钥，不要把真实值提交进仓库。
- `extra_headers` 是为了让 GitHub Copilot 识别请求来源，确保能正确响应。
- 可以增加model_list中的模型来支持多个不同版本的 Claude 模型。

## 启动 LiteLLM

示例命令：

```bash
litellm --config /path/to/config.yaml --port 4000
```

如果你希望长期固定运行，建议直接把端口也固定下来，例如 `4000`。

启动后可以先确认服务是否存活：

```bash
curl -s http://127.0.0.1:4000/v1/models
```

如果服务正常，通常会返回模型列表或鉴权相关响应；如果端口不通，说明 LiteLLM 还没启动成功。

## 首次登录 GitHub Copilot

第一次使用时，需要触发 LiteLLM 走 GitHub OAuth device flow。最直接的方法是发一个最小请求：

```bash
curl http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-local-litellm-key" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [
      {"role": "user", "content": "ping"}
    ]
  }'
```

如果当前没有登录，LiteLLM 通常会提示你完成 GitHub device flow。按提示在浏览器完成授权后，LiteLLM 会缓存 token，后续无需每次重新登录。

建议第一次登录成功后，再执行一次：

```bash
curl -s http://127.0.0.1:4000/v1/models \
  -H "Authorization: Bearer sk-your-local-litellm-key"
```

这样可以更快确认服务和鉴权都已经通了。

## Claude Code 配置

在你的项目或用户目录下配置 `.claude/settings.json`。核心是把 Claude Code 指向本地 LiteLLM：

```json
{
  "env": {
    "ANTHROPIC_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:4000",
    "ANTHROPIC_AUTH_TOKEN": "sk-your-local-litellm-key"
  }
}
```

说明：

- `ANTHROPIC_MODEL` 填你在 LiteLLM 中声明的 `model_name`
- `ANTHROPIC_BASE_URL` 指向本机 LiteLLM 地址
- `ANTHROPIC_AUTH_TOKEN` 对应 LiteLLM 的 `master_key`

配置完成后，再启动 Claude Code 即可。

## 日常使用步骤

如果只追求最短路径，日常使用按下面做：

1. 确认 LiteLLM 在运行。
2. 如果长时间没用过，先用一次 `curl` 测试请求确认登录状态没失效。
3. 进入项目目录。
4. 启动 Claude Code。

最常见的问题其实就两个：

- LiteLLM 根本没启动
- GitHub Copilot 登录态过期，需要重新走一次 device flow

## 个人使用的优化建议

这部分是更实用的重点。因为问题往往不在配置，而在“每次用之前忘了检查服务和登录状态”。

### 建议 1：把 LiteLLM 做成开发机自动服务

在 Linux 开发机上，推荐使用 `systemd --user` 常驻 LiteLLM，而不是每次手动开一个终端。

示例用户服务：

```ini
[Unit]
Description=LiteLLM local gateway for Claude Code
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/litellm --config /home/yourname/.config/litellm/config.yaml --port 4000
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

假设保存为 `~/.config/systemd/user/litellm.service`，常用命令如下：

```bash
systemctl --user daemon-reload
systemctl --user enable --now litellm
systemctl --user status litellm
journalctl --user -u litellm -f
```

这样做的好处是：

- 开机后可自动启动
- 服务异常退出可自动拉起
- 日志集中，排查问题更快

### 建议 2：写一个 Claude Code 启动前检查脚本

如果你不想每次手工判断服务状态，建议写一个简单脚本，在启动 Claude Code 前执行一次。

下面是一个实用的示例：

```bash
#!/usr/bin/env bash
set -euo pipefail

LITELLM_URL="http://127.0.0.1:4000"
LITELLM_KEY="sk-your-local-litellm-key"
MODEL_NAME="claude-sonnet-4-6"

check_port() {
  curl -fsS "$LITELLM_URL/v1/models" \
    -H "Authorization: Bearer $LITELLM_KEY" >/dev/null
}

trigger_login() {
  curl -fsS "$LITELLM_URL/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $LITELLM_KEY" \
    -d "{\"model\":\"$MODEL_NAME\",\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}]}" >/dev/null
}

if ! check_port; then
  echo "LiteLLM 未运行，尝试启动 systemd 用户服务..."
  systemctl --user start litellm
  sleep 2
fi

if ! check_port; then
  echo "LiteLLM 启动失败，请检查 systemctl --user status litellm"
  exit 1
fi

echo "LiteLLM 已启动，检查 GitHub Copilot 登录状态..."
if ! trigger_login; then
  echo "可能需要重新登录 GitHub Copilot，请根据终端提示完成 device flow。"
  exit 1
fi

echo "环境正常，可以启动 Claude Code。"
exec claude
```

这个脚本做了三件事：

1. 检查 LiteLLM 是否在线
2. 如果不在线，就尝试拉起用户服务
3. 用一个最小请求确认当前鉴权是否可用

如果你的 Claude Code 启动命令不是 `claude`，把最后一行替换成你自己的命令即可。

### 建议 3：把“探活”和“重新登录”分开看

很多人会把“服务启动失败”和“GitHub 登录过期”混成一个问题，结果排查很慢。建议固定按下面顺序判断：

1. 端口是否存活
2. `/v1/models` 是否能访问
3. 最小 chat 请求是否成功
4. 如果只有 chat 失败，再怀疑登录态或 provider 兼容性

### 建议 4：只用于个人开发机，不要共享

如果这套方案只是给你自己在开发机上跑，本质上是一个个人工作流优化；但如果你把它做成团队共享网关，风险会明显变大，不建议这么用。

## 常见问题

### Claude Code 启动了，但模型请求失败

优先检查：

- LiteLLM 是否真的在监听 `127.0.0.1:4000`
- `.claude/settings.json` 里的 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 是否跟 LiteLLM 配置一致
- 模型名是否一致，例如 Claude Code 里写的是 `claude-sonnet-4-6`，LiteLLM 配置中也必须是同一个 `model_name`

### 一些 Claude 参数报错

先确认是否开启了 `drop_params: true`。如果没开，Claude Code 传来的额外参数可能直接被后端拒绝。

### 用一段时间后突然不工作

优先怀疑：

- GitHub Copilot 登录态失效
- LiteLLM 进程已经退出
- GitHub Copilot provider 行为变化，导致旧配置失效（需要关注claude code、GitHub Copilot的changelog）