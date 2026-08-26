---
title: Zeno开发进程 -- 给pi编程Agent做一个桌面工作台
description: Zeno是pi编程Agent的桌面壳，记录从v0.1.0到v0.1.5的架构设计、版本演进与踩坑过程，持续更新
categories: [AI, Zeno]
tags: [AI, Zeno, pi, Electron, 编程Agent]
pin: true
---

## 1. 引言

之前在 [终端 AI 编程 Agent 全横评](https://aletheics.github.io/2026/07/31/pi-agent-opencode-crush/) 里梳理过 Pi Agent：Mario Zechner 的极简主义实践，只有 4 个工具、200~300 Token 系统提示词，所有高级能力靠扩展按需安装。极简是优点，但终端形态也确实有天花板——多会话并行看不清状态、长时间任务切走就断、Markdown 和终端混在一个 TTY 里挤。

于是有了 **Zeno**：给 pi 做一个 Codex 风格的桌面壳。核心原则只有一条——

> **不 fork pi，不做第二套配置层。** 配置、包、会话、工具全部留在原生 pi 侧（`~/.pi/agent`），Zeno 只提供界面。
{: .prompt-tip }

代码位于 [aletheics/zeno](https://github.com/aletheics/zeno)，本文记录截至 v0.1.5 的开发进程，后续持续更新。

![Zeno desktop](/images/2026-08-28-zeno-desktop.png)

---

## 2. Zeno 是什么

一句话：**pi coding agent 的桌面外壳**。装了 Zeno 之后，`pi` 命令行该怎么用还怎么用，两边看到的是同一份东西。

几个关键定位：

| 维度 | 说明 |
|------|------|
| 配置来源 | 复用 `~/.pi/agent`（或 `PI_CODING_AGENT_DIR`），模型、API Key、设置、包、工具与交互式 `pi` 完全一致 |
| SDK | 用公开的 `@earendil-works/pi-coding-agent` SDK，不改 pi 源码 |
| userData | Electron 的 `userData` 只放桌面 chrome 偏好（窗口、主题、上次工作区），**绝不**当第二个 agent 配置层 |
| 干净安装 | 全新的 pi home 不会被塞入任何 Zeno 的包、资源或自定义设置 |
| 平台 | Windows / macOS / Linux，三平台安装包 + electron-updater 自动更新 |

对应到界面上，落地的能力大致是：

- **会话不掉线**：切走的会话不会中断，继续在自己的时间线里流式输出
- **状态一目了然**：侧边栏用一个动效标记表达 busy / done / attention 三种运行态
- **终端是一等公民**：内置 `PiTuiTerminal`，pi 的 TUI 能力不被降级
- **项目即主场**：以工作区为单位组织会话，支持 git worktree
- **扩展 UI 可移植**：pi 扩展的 `select` / `confirm` / `input` / `setStatus` / `setWidget` 等映射成桌面原生控件，TUI 独有的能力显式标记为 Degraded 并给出去重的 `unsupported` 诊断——**不静默假装支持**

---

## 3. 架构设计

### 3.1. 进程分层

```text
React Renderer → Preload → Electron Main → utilityProcess Agent Host → pi SDK
```

四层各自的边界很清楚：

1. **Renderer**（React + Vite）：没有任何 Node.js 访问权限，纯 UI。
2. **Preload**：唯一的 IPC 通道，收窄暴露面。
3. **Main**：监督 Agent Host 的生命周期，但**不执行** pi 的工具和扩展。负责窗口、更新、运行时供给这些宿主级的事。
4. **Agent Host**（`utilityProcess`）：真正跑 pi SDK 的地方。

> `utilityProcess` 提供的是**崩溃隔离**，不是安全沙箱。这一点在设计上要写清楚，否则容易误判威胁模型。
{: .prompt-warning }

把 pi 的执行放进独立进程，好处是 agent 跑飞了、扩展炸了，窗口还在，会话状态还能恢复。

### 3.2. 仓库结构

pnpm workspace + `vite-plus`（`vp` 命令）做任务编排：

```text
apps/
  desktop/        # Electron 应用：main / preload / renderer / agent-host / shared
  landing/        # 官网落地页
packages/
  agent-runtime/  # 与 pi 打交道的运行时逻辑（auth/models/session/provider-usage…）
  contracts/      # 跨进程类型契约
  test-utils/     # 测试工具
```

`apps/desktop/src/main` 下的文件名基本能当架构文档看，而且**每个模块都带同名 `.test.ts`**：

```text
pi-sdk.ts           pi-sdk.test.ts           # 解析内置 pi SDK 从哪来
pi-cli-ensure.ts    pi-cli-ensure.test.ts    # 首次启动供给 pi CLI
pi-cli-extract.ts   pi-cli-extract.test.ts   # 解包运行时
host-spawn.ts       host-spawn.test.ts       # 拉起 agent host
host-park-policy.ts host-park-policy.test.ts # 会话驻留策略
auto-update.ts      auto-update.test.ts      # electron-updater
asar-unpack.ts      asar-unpack.test.ts      # 打包时哪些要 unpack
pi-tui-pty.ts       pi-tui-session.ts        # 终端
proxy-discover.ts   proxy-prefs.ts           # 代理
theme-library.ts    shell-path.ts
```

### 3.3. 开发命令

各 app 在仓库根有**独立**的 dev / build 入口：

```bash
pnpm install
pnpm electron:install   # 下载 Electron 43 运行时

pnpm dev                # 桌面端热重载：renderer HMR + 主进程自动重启
pnpm run dev:once       # 一次性冷启动（CI 或不需要 watch 时）
pnpm dev:landing        # 官网 → localhost:5174

pnpm check              # lint + types + format（与 Ubuntu CI 同一套）
pnpm test
pnpm build
pnpm package            # 当前平台安装包 + updater feed
```

`pnpm dev` 会在 `localhost:5173` 起 Vite dev server 给 renderer 做 HMR（改组件即时生效，不重启），同时给 main / preload / agent-host 起 build watcher，后端代码变了才自动重启 Electron。

两个调试用的隔离入口挺好用：

```bash
ZENO_ISOLATED=1 pnpm dev      # 临时 home + fixture 工作区 + 假模型
pnpm demo:session-content     # 纯浏览器预览会话时间线（不起 Electron）→ 127.0.0.1:4177
```

---

## 4. 版本演进（v0.1.0 → v0.1.5）

从 2026-08-11 到 2026-08-26，65 个 commit、6 个 tag、约 16 天。

### 4.1. v0.1.0（08-11）— 更名与地基

项目原名 **Pix**，v0.1.0 前整体更名为 **Zeno**（代码、landing、LICENSE、测试断言、AUMID 品牌标识全量替换）。同期做完的地基工作：

- 加入 watch 模式与 HMR，Windows 下用 `taskkill` 正确收尾、避免重复开窗
- 全量补 `cursor-pointer`（侧边栏、命令面板、composer、模型菜单、Switch……）——这类细节堆起来才有"原生桌面应用"的手感
- 非 git 工作区下 git-branches 优雅降级
- 重新生成多分辨率 ICO

### 4.2. v0.1.1（08-12）— MCP 与发布流水线

- **MCP server 管理** + `/mcp` 斜杠命令，Windows spawn 修复
- CI 里从 git tag 自动写入应用版本
- NSIS 关掉 solid 压缩，安装解压更快

### 4.3. v0.1.2（08-14）— 斜杠命令、流式与安全加固

这一版内容最密集：

**新增**

- `/login` 内置命令，直接打开 provider 登录页
- 批量导入自定义模型：拉取 provider 的模型列表后一次性添加
- **桌面内置命令与 AI 路由命令分离**——输 builtin 命令走本地执行，不再被当成 prompt 发给模型

**修复**

- 斜杠命令 / prompt 模板 / skill 只显示你输入的 `/skill:name`，展开后的 `<skill>` 正文不再泄漏成用户消息刷进时间线
- **流式重新变顺滑**：Markdown 重解析限流到约 10 次/秒、延迟上界 100ms，替换掉原来 `useDeferredValue` 的无界滞后（那种"先卡住再一次性倾泻"的手感）
- subagent 前台 spawn 在内置二进制不可用时回退到 Node

**安全**

- 桌面壳加固：命令注入、路径穿越、不安全 IPC

### 4.4. v0.1.3（08-15）— 打包瘦身与 Windows

- 退出后重启，自动重开上次的前台会话
- pi 运行时升级到 0.84.2
- 打包只 unpack 原生模块；内置 pi CLI 改为**首次启动时**解包到 `userData/pi-cli`，不再直接塞一个未压缩的 `node_modules`
- Windows：运行时解包改用系统 `tar`；打包时断言内置运行时确实存在

### 4.5. v0.1.4（08-18）— 会话驻留

- **Parked sessions 成为一等公民**：忙碌的 agent host 永不被驱逐，空闲的保温 10 分钟；切走时 parked run 继续流进自己的会话而不是被 abort
- 侧边栏运行态标记换成随状态形变的动效标记，替掉 spinner 和那个半圆"等待"字符
- Linux：可拖拽标题栏不再抢走窗口按钮的点击（把 caption buttons portal 到 body）

### 4.6. v0.1.5（08-26）— 两个隐蔽 bug

- pi 运行时升级到 0.84.3
- 启动时打印**解析到的内置 pi SDK 及其来源路径**——运行时版本不对能立刻看见，而不是等到出问题才发现
- 修掉下面两个 bug（值得单独说）

---

## 5. 踩坑记录

### 5.1. 字号设置改了但 UI 不动

**现象**：Settings → Appearance → Typography 改字号，界面纹丝不动。

**原因**：字号 token 既声明在 `:root`，又在各主题块里重复声明了一遍。而 app shell 自己带着 `data-theme`，于是主题块的选择器在整个 shell 子树里**赢过了**写在 `<html>` 上的值——不管设置成什么，全都被钉死在 14px / 12px 的默认值上。

**教训**：CSS 变量的"最近祖先胜出"直觉，会被同一元素上更高特异性的选择器打破。主题块只该定义颜色，尺寸类 token 不要跟着复制一份。

### 5.2. 升级后还在跑旧版 pi

**现象**：应用升级了，跑的还是旧版 pi。

**原因**：`userData/pi-cli` 只要**目录存在**就被当成内置 SDK 的有效来源，而它在搜索顺序里排在 `node_modules` 前面。于是早前版本留下的一次解包，悄悄把应用钉死在那个旧 pi 上。

**修复**：只有当 `userData/pi-cli` 的版本**与本次实际打包的版本一致**时才采信；开发环境下彻底不看它——`node_modules` 是唯一事实来源。

> 缓存目录当"存在即有效"用，是个反复出现的坑。带上版本指纹，或者干脆每次校验。
{: .prompt-warning }

### 5.3. 其他

- pi-coding-agent 的文件锁竞争，用 `pnpm patch` 打了补丁
- dev watch 下 Ctrl+C 会重复弹终止确认

---

## 6. CI 与发布

| 工作流 | 文件 | 触发 | 内容 |
|--------|------|------|------|
| **CI** | `.github/workflows/ci.yml` | PR + push `main` | Ubuntu：install → lint/types/format → tests → build |
| **Release** | `.github/workflows/release.yml` | push `v*` tag（或手动） | 多平台安装包 + updater feed → GitHub Release |

### 6.1. 版本号只有一个来源：git tag

产品版本在**构建时**从 tag 推导——Release 工作流剥掉 `v` 前缀写进 `apps/desktop/package.json`，所以 tag 和安装包版本**不可能漂移**。仓库里 checked-in 的那个版本号只是本地开发的 fallback，发布时不要手动 bump。根包和 `packages/*` 恒为 `0.0.0`（私有 workspace 包）。

```bash
git tag v0.1.0
git push origin v0.1.0
```

这个设计值得抄：**版本号有且只有一个事实来源**，人手同步两处迟早出错。

### 6.2. 发布产物

每个 tag 只发安装器和 electron-updater 真正需要的东西：

| 产物 | 作用 |
|------|------|
| `Zeno-*-win-x64.exe` | Windows 安装（NSIS） |
| `latest.yml` | Windows 更新 feed |
| `Zeno-*-mac-{arm64,x64}.dmg` | macOS 手动安装 |
| `Zeno-*-mac-{arm64,x64}.zip` | macOS **自动更新**载荷 |
| `latest-mac.yml` | macOS 更新 feed（同时列出两个 zip） |
| `Zeno-*-linux-*.AppImage` | Linux 运行 / 更新 |
| `Zeno-*-linux-*.deb` | Linux 手动安装 |
| `latest-linux.yml` | Linux 更新 feed |
| `*.blockmap` | 差分下载映射 |

**任一必需的 feed 或 mac zip 缺失，CI 直接失败**（`scripts/release-assets.mjs`）。这条硬校验很关键——updater feed 缺了，用户侧表现是"永远收不到更新"，而不是构建报错，属于最难发现的那类故障。

### 6.3. 签名

目前**未签名**（打包时设 `CSC_IDENTITY_AUTO_DISCOVERY=false`）。

> macOS 首次打开未签名的下载可能需要 `xattr -cr /Applications/Zeno.app` 解除 Gatekeeper 隔离。
>
> 但**自动更新不需要** Apple Developer ID——Zeno 通过 electron-updater 的 `sha512` 校验 release zip 后自行替换 `.app`，跟 Tauri updater + minisign 是同一个模型。配上 `CSC_LINK` / `CSC_KEY_PASSWORD` 只是让 Gatekeeper 体验和通知更好。
{: .prompt-info }

---

## 7. 小结与后续

回头看，这个项目真正做对的几个决定：

1. **不 fork，只做壳**。所有配置留在 `~/.pi/agent`，桌面端和 CLI 共享同一份状态。代价是不能随便改 pi 的行为，收益是 pi 每次升级都能直接吃到。
2. **进程边界写死在文档里**。"Renderer 无 Node""Main 不执行 pi 工具""userData 不是第二配置层"——这几条写进 README，评审时就有依据。
3. **版本号单一来源**。tag 驱动，人手改不了。
4. **发布产物做硬校验**。缺 feed 就 fail，不给静默降级的机会。
5. **模块与测试同名并列**。`pi-sdk.ts` / `pi-sdk.test.ts` 这种排布，新人扫一眼目录就知道系统由哪些关注点组成。

参考链接：

* [Zeno 仓库](https://github.com/aletheics/zeno)
* [pi coding agent](https://pi.dev)
* [终端 AI 编程 Agent 全横评：Pi / Oh My Pi / OpenCode / Crush](https://aletheics.github.io/2026/07/31/pi-agent-opencode-crush/)
* `packages/agent-runtime/EXTENSION_UI.md` — 扩展 UI 的完整映射表
