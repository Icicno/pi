<p align="center">
  <a href="https://pi.dev">
    <img alt="pi logo" src="https://pi.dev/logo-auto.svg" width="128">
  </a>
</p>

# Pi Agent Harness

这是 Pi agent harness 项目的主仓库，包含可自行扩展的编码代理。

* **[@earendil-works/pi-coding-agent](packages/coding-agent)**：交互式编码代理 CLI
* **[@earendil-works/pi-agent-core](packages/agent)**：支持工具调用和状态管理的代理运行时
* **[@earendil-works/pi-ai](packages/ai)**：统一的多提供商 LLM API（OpenAI、Anthropic、Google 等）

了解 Pi 的更多信息：

* [访问 pi.dev](https://pi.dev)，查看项目网站和演示
* [阅读文档](https://pi.dev/docs/latest)，也可以直接让代理解释自身

## 所有软件包

| 软件包 | 说明 |
|---------|-------------|
| **[@earendil-works/pi-ai](packages/ai)** | 统一的多提供商 LLM API（OpenAI、Anthropic、Google 等） |
| **[@earendil-works/pi-agent-core](packages/agent)** | 支持工具调用和状态管理的代理运行时 |
| **[@earendil-works/pi-coding-agent](packages/coding-agent)** | 交互式编码代理 CLI |
| **[@earendil-works/pi-tui](packages/tui)** | 支持差分渲染的终端 UI 库 |

有关 Slack/聊天自动化和工作流，请参阅 [earendil-works/pi-chat](https://github.com/earendil-works/pi-chat)。

## 权限与容器化

Pi 不包含用于限制文件系统、进程、网络或凭据访问的内置权限系统。默认情况下，它使用启动 Pi 的用户和进程所拥有的权限运行。

如果需要更强的边界，请将 Pi 放入容器或沙箱中。参阅 [packages/coding-agent/docs/containerization.md](packages/coding-agent/docs/containerization.md) 中介绍的三种模式：

* **Gondolin 扩展**：让 `pi` 和提供商认证保留在宿主机，同时将内置工具和 `!` 命令路由到本地 Linux micro-VM。
* **纯 Docker**：将整个 `pi` 进程运行在本地容器中，提供简单隔离。
* **OpenShell**：将整个 `pi` 进程运行在策略控制的沙箱中。

## 贡献

贡献指南见 [CONTRIBUTING.md](CONTRIBUTING.md)，项目规则见 [AGENTS.md](AGENTS.md)（同时适用于人类和代理）。Pi 的长期规划也可以在 [RFC](https://rfc.earendil.com/keyword/pi/) 中找到。

## 开发

```bash
npm install --ignore-scripts  # 安装所有依赖，但不运行生命周期脚本
npm run build                 # 构建所有软件包
npm run check                 # 运行 lint、格式化和类型检查
./test.sh                     # 运行测试（没有 API 密钥时跳过依赖 LLM 的测试）
./pi-test.sh                  # 从源码运行 pi（可从任意目录执行）
```

## 供应链加固

我们将 npm 依赖变更视为需要审查的代码变更。

* 直接外部依赖固定到精确版本；内部 workspace 软件包仍使用版本范围。
* `.npmrc` 设置 `save-exact=true` 和 `min-release-age=2`，避免 npm 解析时使用当天发布的依赖。
* `package-lock.json` 是依赖事实来源。提交前检查会阻止意外提交锁文件，除非设置 `PI_ALLOW_LOCKFILE_CHANGE=1`。
* `npm run check` 会验证直接依赖是否固定、原生 TypeScript import 是否兼容，以及生成的 coding-agent shrinkwrap。
* 发布的 CLI 软件包包含 `packages/coding-agent/npm-shrinkwrap.json`，它从根目录锁文件生成，用于固定 npm 用户的传递依赖。
* 发布 smoke test 使用 `npm run release:local`，在打 tag 前构建、打包，并在仓库外创建隔离的 npm 和 Bun 安装。
* 本地发布安装、文档中的 npm 安装以及 `pi update --self` 在支持时使用 `--ignore-scripts`。
* CI 使用 `npm ci --ignore-scripts` 安装，并由定时 GitHub workflow 执行 `npm audit --omit=dev` 和 `npm audit signatures --omit=dev`。
* shrinkwrap 生成对依赖生命周期脚本使用显式允许列表；新增此类依赖必须经过审查，否则检查失败。

## 分享你的开源编码代理会话

如果你使用 Pi 或其他编码代理进行开源工作，请分享你的会话。

公开的开源会话数据可以用真实任务、工具使用、失败和修复来改进编码代理，而不是只依赖玩具基准。

完整说明见 [这篇 X 文章](https://x.com/badlogicgames/status/2037811643774652911)。发布会话请使用 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)，并阅读其中的 README.md 获取设置说明。你只需要 Hugging Face 账号、Hugging Face CLI 和 `pi-share-hf`。

## 许可证

MIT
