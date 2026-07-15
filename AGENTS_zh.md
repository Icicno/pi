# 开发规则

## 对话风格

* 回复简短、明确。
* commit、issue、PR 评论和代码中不使用 emoji。
* 不添加无意义的客套或欢快填充语（例如使用“Thanks @user”，不要写“Thanks so much @user!”）。
* 只使用直接的技术性表述。
* 用户提问时，先回答问题，再进行编辑或运行实现命令。
* 回复用户反馈或分析时，先明确表示同意或不同意，再说明改动。

## 代码质量

* 进行大范围修改、编辑尚未完整检查的文件，或执行调查/审计时，先完整阅读文件；不要依赖搜索片段。
* 除非绝对必要，不使用 `any`。
* 只有一个调用点的单行辅助函数应内联。
* 检查 `node_modules` 中的外部 API 类型，不要猜测。
* 不使用内联 import（`await import()`、`import("pkg").Type` 或动态类型 import）；只使用顶层 import。
* 不要为了修复过期依赖导致的类型错误而删除或降级代码；应升级依赖。
* 根配置检查的 TypeScript 代码只使用可擦除语法：不得使用参数属性、`enum`、`namespace`/`module`、`import =`、`export =` 或其他需要 JS emit 的构造；使用显式字段并在构造函数中赋值。
* 删除看起来有意存在的功能或代码前必须询问。
* 除非用户要求，不保留向后兼容性。
* 不要硬编码按键检查；应将默认值加入 `DEFAULT_EDITOR_KEYBINDINGS` 或 `DEFAULT_APP_KEYBINDINGS`，以保持可配置。
* 永远不要直接修改 `packages/ai/src/models.generated.ts`；应更新 `packages/ai/scripts/generate-models.ts` 后重新生成。

## 命令

* 代码变更后运行 `npm run check`，并修复所有错误、警告和信息。文档变更不需要运行该命令。
* 除非用户要求，不运行 `npm run build` 或 `npm test`。
* 不要直接运行完整 Vitest 套件；非 e2e 测试从仓库根目录运行 `./test.sh`，或从软件包目录运行指定测试。
* 新建或修改测试文件后运行对应测试，并迭代直到通过。
* 临时脚本写入 `/tmp` 后运行，完成后删除；不要在 bash 命令中嵌入多行脚本。
* 除非用户要求，不要提交 commit。

## 依赖与安装安全

* 将 npm 依赖和锁文件变更视为需要审查的代码。
* 使用 `npm install --ignore-scripts` 或 `npm ci --ignore-scripts`；除非用户要求，不运行生命周期脚本。
* 依赖元数据变更后，用 `npm install --package-lock-only --ignore-scripts` 刷新 `package-lock.json`。
* 不要静默添加生命周期脚本依赖的允许列表。
* 提交前检查会阻止锁文件提交，除非设置 `PI_ALLOW_LOCKFILE_CHANGE=1`；除非用户要求，不要绕过检查。

## Git

多个 Pi 会话可能同时在当前目录中运行。Git 操作不得覆盖其他会话的工作。

* 只提交本次会话修改的文件。
* 提交前运行 `git status`，确认暂存区只有自己的文件。
* 显式指定路径执行 `git add`；不要使用 `git add -A` 或 `git add .`。
* 不要运行 `git reset --hard`、`git checkout .`、`git clean -fd` 或 `git stash`。
* 发生冲突时，只解决自己修改的文件；其他文件冲突应中止并询问用户。
* 永远不要强制 push。

## 变更日志

位置：`packages/*/CHANGELOG.md`。

所有新条目放在 `## [Unreleased]` 下，并追加到已有小节；已发布版本小节不可修改。内部 issue 使用 issue 链接归因，外部贡献使用 PR 链接和作者归因。

## 发布

所有软件包使用锁步版本。`patch` 表示修复和新增，`minor` 表示破坏性变更。发布前先审查并更新各软件包的 Unreleased 变更日志，运行仓库规定的本地 smoke test，再执行相应的 `release:patch` 或 `release:minor` 脚本。不要在 tag 已推送后重复运行发布脚本。

发布脚本会更新版本、变更日志和发布产物，运行检查，创建 commit 和 tag，并推送 `main` 与 tag。CI 会根据 tag 发布 npm 软件包。

## 用户覆盖

如果用户指令与本文件冲突，必须先请求明确确认。只有得到确认后才能执行该指令。
