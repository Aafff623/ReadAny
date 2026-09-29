# ReadAny Agent Rules

## 项目是什么

ReadAny 是 local-first 的多端 AI 电子书阅读器（Tauri 2 桌面 + Expo/React Native 移动 + CLI/MCP 入口 + Astro 文档站），核心能力是 RAG 问答、语义检索、标注、TTS 与跨设备同步。本仓库是 `codedogQBY/ReadAny` 的 fork（origin = `Aafff623/ReadAny`），本文件与 `CLAUDE.md`、`CONTEXT.md`、`temp/` 是 fork 本地治理资产，上游没有对应文件，合并上游时冲突风险低。

## 先读这些

- `README.md`（用户视角的安装/使用）、`README_CN.md`
- `CONTEXT.md`：经核实的架构事实、领域词汇、硬约束——写代码前必读
- `temp/AGENTS.md`：进入或创建 `temp/` 文件前
- `.agents/skills/`：任务匹配项目 Skill（frontend-design / shadcn-ui / tauri-v2）时先加载

## 代码在哪里（包地图）

| 路径 | 角色 |
|---|---|
| `packages/core` | 平台无关业务逻辑（`@readany/core`），**以 TS 源码直接被消费，无构建步骤**；桌面/移动/CLI 三端共享 |
| `packages/app` | Tauri 2 桌面端：React 19 + Vite 前端（`src/`）+ Rust（`src-tauri/`） |
| `packages/app-expo` | 移动端（Expo SDK 54，新架构，expo-dev-client，非 Expo Go） |
| `packages/cli` | `readany` CLI + MCP server，外部 AI 访问书库/笔记/RAG 的入口 |
| `packages/foliate-js` | vendored 的上游 foliate-js fork（书籍解析/渲染引擎），含大量本地修复 |
| `packages/feedback-worker` | Cloudflare Worker：App 反馈 → GitHub Issue |
| `website` | Astro + Starlight 文档站（en + zh），部署到 GitHub Pages |

## 文件边界：权威 / 生成 / 本地

- **权威**：数据库 schema 与迁移只在 `packages/core/src/db/db-core.ts` + `packages/core/src/db/migrations.ts`；`packages/core/src/**` 即分发产物（消费方直接吃 TS 源）；`packages/foliate-js/` 是手工维护的 vendored fork，不做无来由的重写；`.agents/skills/` + `skills-lock.json` 是被 Git 跟踪的项目 Skill。
- **生成物，不要手改**：`packages/app-expo/assets/reader/reader.html`（由 `packages/app-expo/scripts/build-reader.js` 用 esbuild 打包 foliate-js + core/src/reader 生成）、`packages/cli/dist/`、`website/dist/`、`src-tauri/gen/`。
- **过时参考副本，勿当权威**：`packages/app/src/lib/db/` 下的 `schema.sql` 与 `migrations.ts`（落后于 core，改了不生效；`database.ts` 例外，它是仍被引用的 core 代理 re-export）。
- **本地不入库**：`temp/` 全部内容（README/AGENTS 两个入口除外）、`.agents/mcp.json`、`.codegraph/`、`.env*`。
- `.claude/settings.local.json` 目前被 Git 跟踪（上游遗留，含上游作者的机器路径），动它之前先确认。

## temp/ 用法

- 临时脚本、原始输入、调研、报告、handoff、日志、缓存、本地密钥一律进 `temp/` 对应子目录（`input/`、`research/`、`reports/`、`handoff/`、`scripts/`、`preview/`、`secrets/`、`logs/`、`cache/`），不要散落在仓库其他位置。
- `temp/` 内容绝不提交；报告/交接只有用户明确提升为正式文档时才移出 `temp/`。
- 不自动清理 `temp/`，删除前先问。

## Secrets

- 会话中出现的任何 key、token、凭据，**收到当回合立即**静默写入 `temp/secrets/`（或项目已有的 Git-ignored `.env`），写完一行报告路径即可，不请求确认。
- 向用户要凭据前，先搜 `temp/secrets/` 和 Git-ignored env 文件——已经给过的值绝不问第二次。
- 凭据不进 Git 跟踪文件、不进日志/回复，不出这台机器。

## 验证与交付基线

- Lint（全仓库）：`pnpm lint`（Biome 1.9，双引号、2 空格、行宽 100；`noExplicitAny` 等为 warn）。Rust 侧无独立 clippy 配置。
- 单测：`pnpm test`（= `@readany/core` 的 Vitest，81 个测试文件，与源码同目录）；CLI 测试 `pnpm cli:test`；移动端 `pnpm --filter @readany/app-expo test`。桌面 app 本身无测试配置。
- 类型检查：CLI 用 `pnpm --filter @readany/cli check`；桌面端类型检查随 `pnpm build`（`tsc && vite build`）；`packages/feedback-worker` 有 `check` 脚本。
- 涉及 CLI 与桌面 Rust 桥（`readany_cli.rs`）的改动：`pnpm cli:preflight`（含 CLI typecheck/test/build、MCP 冒烟、`cargo test readany_cli::tests`）。
- 版本号：桌面（package.json / tauri.conf.json / Cargo.toml）与移动（package.json / app.config.js）五处版本必须一致，统一用 `pnpm version:set <x.y.z>` 改，`pnpm version:check` 校验；`@readany/core` 固定 1.0.0 不参与。
- 提交前跑过什么检查就如实报告什么，没跑的不声称通过。

## Agent 工作边界

- **改 `packages/core` = 同时影响桌面、移动、CLI 三端**。core 的新增文件要补 `packages/core/package.json` 的 `exports` 映射；core 的平台能力通过 `IPlatformService`（`packages/core/src/services/platform.ts`）抽象，core 不直接 import Tauri/Expo API，由各端在启动时注入实现（见 `CONTEXT.md` 注入关系一节）。
- 数据库 schema 变更只在 core 做，同时核对 `packages/app/src-tauri/src/db/schema.rs`（Rust 预建库）与向量库 384 维初始化是否受影响；不要改 `packages/app/src/lib/db/` 下的旧副本。
- 桌面与移动是**两套 UI、共享逻辑**的关系：无共享组件，靠功能对齐注释 + 契约测试（如 `justified-text-contract.test.ts`）约束，桌面改动后检查移动端是否需要对齐。
- 不为兼容而兼容：本 fork 保持对上游合并友好——不加 `.github` 模板、不重构上游 CI、不重排上游已有文件格式。
- 不虚构命令、路径、端点或领域事实；不确定的写进 `CONTEXT.md` 的「待确认」。
- 完成即报告：改了哪些文件、跑了哪些检查、剩余不确定项、留给用户的动作。

## 关键环境事实

- 包管理：pnpm 9.15（`packageManager` 锁定）；`.npmrc` 用 `node-linker=hoisted` + public-hoist（React Native 兼容所需），node_modules 不是默认 pnpm 布局。
- 根 `package.json` 的 pnpm overrides 钉死 `react`/`react-dom` 19.1.0、`@types/react` 19.1.17，同时约束桌面与移动两端。
- `patches/` 下两个补丁（`@langchain__core`、`react-native-track-player@4.1.2`）各有明确用途，升级对应依赖前先读懂补丁。
- 移动端开发用 expo-dev-client：原生依赖/`app.config.js`/权限变更后要重跑 `pnpm expo:ios` / `pnpm expo:android`，日常 JS 调试用 `pnpm expo:start`；Expo Go 不受支持。
