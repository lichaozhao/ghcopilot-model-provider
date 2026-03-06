# GitHub Copilot LiteLLM 本地代理 PRD

## 1. 目标

提供一个本地工具，把 GitHub Copilot 模型通过 LiteLLM 暴露为稳定的本地代理服务，并向用户输出可直接接入第三方客户端的连接信息。

目标结果：

1. 用户可以选择若干 GitHub Copilot 模型并映射为本地代理名称。
2. 工具可以生成最小 LiteLLM 配置并以长期服务方式运行。
3. Claude 系列模型可被 Claude Code 以 Anthropic-compatible 方式接入。
4. GPT / Codex 系列模型可被 OpenAI-compatible 客户端接入。
5. 用户可以明确看到代理地址、认证 key、原始模型名和代理名映射。
6. 首次认证和后续重新认证有清晰的前台入口和状态反馈。

## 2. 目标用户

1. 已有 GitHub Copilot 订阅的开发者。
2. 希望在本机把 GitHub Copilot 模型复用到 Claude Code 的用户。
3. 希望在本机把 GitHub Copilot 模型复用到 Codex CLI、Codex App 或其他 OpenAI-compatible 客户端的用户。

## 3. 关键用户故事

### 用户故事 1：初始化配置

作为用户，我希望通过一次交互式配置选择要暴露的模型和代理名，而不是手工编写 LiteLLM 配置。

### 用户故事 2：接入 Claude Code

作为用户，我希望拿到一组可直接填给 Claude Code 的接入参数，而不需要理解 LiteLLM 的内部配置。

### 用户故事 3：接入 OpenAI-compatible 客户端

作为用户，我希望拿到一组可直接填给 OpenAI-compatible 客户端的接入参数，包括地址、key 和模型名。

### 用户故事 4：长期运行

作为用户，我希望 LiteLLM 代理能长期运行，不需要每次手工启动。

### 用户故事 5：首次认证

作为用户，我希望在首次使用时，有一个明确的命令能触发 GitHub Copilot device flow，并在前台把验证码和授权地址展示出来。

### 用户故事 6：认证失效恢复

作为用户，我希望在认证失效时，能通过明确的状态命令知道需要重新登录，而不是只看到客户端请求失败。

## 4. 产品范围

### 4.1 交互式配置

工具必须提供交互式初始化流程，至少包含以下输入项：

1. 选择需要暴露的模型。
2. 为每个模型指定代理名称。
3. 指定代理监听地址和端口。
4. 指定或自动生成 LiteLLM master key。
5. 指定服务运行方式所需的本地路径。

初始化完成后，工具必须生成可直接使用的本地配置产物。

### 4.2 内置模型目录

工具必须内置一份可直接使用的 GitHub Copilot 模型目录，首版允许硬编码。

每个模型条目必须包含：

1. 展示名称
2. LiteLLM 底层模型名
3. 默认代理名称
4. 客户端兼容类型
5. 是否需要 Responses API

首版最少支持：

1. Claude Sonnet 4.6
2. Claude Opus 4.6
3. GPT-5.3-Codex
4. GPT-5.4

### 4.3 配置生成

工具必须生成最小可运行的 LiteLLM 配置文件。

要求：

1. 只包含运行本地代理所需内容。
2. 支持 Claude 模型与 OpenAI-compatible 模型并存。
3. 对需要 Responses API 的模型自动加正确配置。
4. 配置文件和生成的 key 存放在用户配置目录，而不是仓库目录。

### 4.4 服务管理

工具必须支持将 LiteLLM 以本地长期服务方式运行。

Linux 首版以 `systemd --user` 为标准方案。

必须提供以下能力：

1. 安装服务
2. 启动服务
3. 停止服务
4. 重启服务
5. 查看状态
6. 查看日志

### 4.5 接入信息输出

工具必须输出一份可直接使用的连接信息。

输出内容必须包含：

1. 代理地址
2. LiteLLM 认证 key
3. 原始模型名
4. 代理模型名
5. Claude Code 所需环境变量
6. OpenAI-compatible 客户端所需环境变量

输出形式至少包括：

1. 终端可读说明
2. 可直接复制的环境变量片段

### 4.6 Claude Code 兼容性

Claude 系列模型必须以 Claude Code 可直接接入的形式暴露。

必须满足：

1. 代理地址可作为 Anthropic-compatible endpoint。
2. 模型别名可作为 Claude Code 的模型名。
3. 输出中必须提供：
   - `ANTHROPIC_BASE_URL`
   - `ANTHROPIC_AUTH_TOKEN`
   - `ANTHROPIC_MODEL`

### 4.7 OpenAI-compatible 兼容性

GPT / Codex 系列模型必须以 OpenAI-compatible 形式暴露。

输出中必须提供：

1. `OPENAI_BASE_URL`
2. `OPENAI_API_KEY`
3. `OPENAI_MODEL`

### 4.8 认证体验

工具使用 LiteLLM 默认 GitHub Copilot device flow 作为认证方式。

首版不要求后台服务自动弹出交互登录界面，但必须提供两个前台命令：

1. `login`
2. `status` 或 `doctor`

`login` 的要求：

1. 主动触发一次最小请求，命中 `github_copilot/...` 模型。
2. 在前台把 LiteLLM 输出的 device code、授权地址和提示信息展示给用户。
3. 登录成功后返回明确成功状态。

`status` 或 `doctor` 的要求：

1. 显示 LiteLLM 服务是否运行。
2. 显示当前配置的代理地址和模型映射。
3. 检查认证缓存是否存在。
4. 在认证失效时明确返回 `reauth_required` 或等价状态。

### 4.9 认证缓存

工具必须管理并展示 LiteLLM 使用的 GitHub Copilot 认证缓存位置。

要求：

1. 支持默认缓存目录。
2. 支持自定义缓存目录。
3. 用户可通过工具查看当前缓存目录位置。
4. 用户可通过工具重新触发认证，而不需要手工理解缓存文件格式。

## 5. 典型用户流程

### 流程 A：首次配置

1. 用户执行 `setup`。
2. 工具展示可用模型列表。
3. 用户选择模型并设置代理名。
4. 工具生成 LiteLLM 配置和本地 key。
5. 工具安装并启动本地服务。
6. 工具输出接入信息。

### 流程 B：首次登录

1. 用户执行 `login`。
2. 工具触发最小模型请求。
3. 前台展示 GitHub device code 和授权地址。
4. 用户在浏览器完成授权。
5. 工具提示登录成功。

### 流程 C：接入客户端

1. 用户执行 `print-claude-env` 或 `print-openai-env`。
2. 工具输出所需环境变量。
3. 用户将这些参数配置到自己的客户端。

### 流程 D：认证失效恢复

1. 用户执行 `status`。
2. 工具发现认证失效并显示需要重新登录。
3. 用户执行 `login` 重新认证。

## 6. 功能模块拆分

### 模块 A：模型目录

负责内置模型元数据与默认别名。

### 模块 B：配置向导

负责收集用户输入并生成配置文件。

### 模块 C：服务管理

负责安装和控制本地 LiteLLM 服务。

### 模块 D：认证控制

负责显式登录、认证状态检查和失效提示。

### 模块 E：连接信息输出

负责输出模型映射、代理地址、key 和客户端环境变量。

## 7. 命令面设计

首版建议提供以下命令：

1. `setup`
2. `start`
3. `stop`
4. `restart`
5. `status`
6. `logs`
7. `login`
8. `print-models`
9. `print-claude-env`
10. `print-openai-env`

## 8. 验收标准

### 功能验收

1. 用户可以通过交互式流程选择模型并生成配置。
2. 用户可以安装并启动本地 LiteLLM 长期服务。
3. 用户可以看到原始模型名与代理名映射。
4. 用户可以获得 Claude Code 所需的接入参数。
5. 用户可以获得 OpenAI-compatible 客户端所需的接入参数。
6. 用户可以通过 `login` 命令完成首次 GitHub Copilot device flow 登录。
7. 用户可以通过 `status` 命令发现认证失效并得到重新登录提示。

### 体验验收

1. 用户不需要手工写 LiteLLM 配置。
2. 用户不需要手工查找 token 缓存目录。
3. 用户不需要手工解析 LiteLLM 日志才能完成首次登录。
4. 用户不需要理解底层 `github_copilot/...` 细节即可完成客户端接入。

## 9. 约束

1. 认证方式以 LiteLLM 默认 GitHub Copilot device flow 为准。
2. 首版模型目录允许硬编码。
3. 首版 Linux 以 `systemd --user` 作为标准长期服务实现。
4. Claude 模型的代理输出必须兼容 Claude Code。