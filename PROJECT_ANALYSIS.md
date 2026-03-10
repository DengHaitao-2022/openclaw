# OpenClaw 项目详细解析

> 本文档对 OpenClaw 开源项目进行全面的技术解析，涵盖项目架构、核心模块、数据流、扩展机制与开发指南。

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈](#2-技术栈)
3. [顶层目录结构](#3-顶层目录结构)
4. [核心架构](#4-核心架构)
5. [源码模块详解（src/）](#5-源码模块详解src)
6. [原生应用（apps/）](#6-原生应用apps)
7. [扩展/插件系统（extensions/）](#7-扩展插件系统extensions)
8. [文档结构（docs/）](#8-文档结构docs)
9. [测试基础设施](#9-测试基础设施)
10. [构建与部署](#10-构建与部署)
11. [消息数据流](#11-消息数据流)
12. [开发工作流](#12-开发工作流)

---

## 1. 项目概述

**OpenClaw** 是一个运行在用户自有设备上的**个人 AI 助手平台**。它通过用户日常使用的即时通信渠道（如 WhatsApp、Telegram、Discord、iMessage 等）响应用户请求，并提供语音交互、画布渲染等高级能力。

### 核心特性

| 特性 | 说明 |
|------|------|
| 多渠道支持 | 20+ 主流消息平台（WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等） |
| 自托管 | Gateway（网关）运行在用户设备上，数据不经过第三方 |
| 多模型支持 | OpenAI、GitHub Copilot、Google Gemini、Qwen、MiniMax 等 |
| 插件系统 | 40+ 官方扩展，支持社区自定义插件 |
| 多平台应用 | iOS、Android、macOS 原生应用 |
| 语音交互 | macOS/iOS/Android 上支持语音输入和 TTS |
| Canvas 渲染 | 实时可编辑 HTML/CSS/JS 画布 |
| OpenAI 兼容 API | 可作为 OpenAI 兼容的 API 服务端 |

### 版本信息

- 当前版本：`2026.3.9`（以 `package.json` 中的 `version` 字段为准）
- 代码规模：约 617,000+ 行 TypeScript（截至 v2026.3.9 的估算值，会随版本持续增长）
- 测试覆盖：600+ 测试文件（截至 v2026.3.9，随开发持续增加）

---

## 2. 技术栈

### 后端 / CLI

| 技术 | 用途 |
|------|------|
| **Node.js 22+** | 运行时基础（生产环境） |
| **TypeScript (ESM, strict)** | 主要编程语言 |
| **Bun** | 开发/测试时的 TS 直接执行 |
| **pnpm** | 包管理（工作区模式） |
| **Commander.js** | CLI 框架 |
| **tsdown** | 构建打包 |
| **Vitest** | 单元/集成测试框架 |
| **Oxlint / Oxfmt** | 代码检查与格式化 |

### 原生应用

| 平台 | 技术 |
|------|------|
| iOS | Swift + SwiftUI + Observation |
| macOS | Swift + SwiftUI + Observation |
| Android | Kotlin + Gradle |

### 基础设施

| 技术 | 用途 |
|------|------|
| Docker | 容器化部署 |
| Fly.io | 云端部署（fly.toml） |
| Render | 备选云部署（render.yaml） |
| WebSocket | Gateway ↔ 客户端实时通信 |
| TypeBox | 协议 Schema 定义与 JSON Schema 生成 |

---

## 3. 顶层目录结构

```
openclaw/
├── src/                    # 核心 TypeScript 源码（55+ 子目录，617K+ 行）
├── apps/                   # 原生应用（iOS / Android / macOS）
├── extensions/             # 官方扩展/插件（40 个）
├── docs/                   # 文档站点（Mintlify，含 zh-CN / ja-JP 翻译）
├── ui/                     # Web 控制界面前端组件
├── test/                   # 集成 & E2E 测试
├── test-fixtures/          # 测试用固定数据/资源
├── skills/                 # 内置 Agent 技能
├── packages/               # Workspace 内部包
├── scripts/                # 构建、发布、辅助脚本
├── vendor/                 # 已 vendor 化的第三方代码
├── assets/                 # 静态资源（Logo 等）
├── openclaw.mjs            # CLI 入口（Node.js bootstrap）
├── package.json            # 项目配置与脚本
├── pnpm-workspace.yaml     # pnpm 工作区配置
├── tsconfig.json           # TypeScript 配置
├── tsdown.config.ts        # 构建配置
├── Dockerfile              # 多阶段 Docker 构建
├── fly.toml                # Fly.io 部署配置
├── vitest.config.ts        # 主测试配置（+8 个专用配置）
└── CHANGELOG.md            # 版本变更日志
```

---

## 4. 核心架构

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     消息渠道（20+ 平台）                          │
│  WhatsApp  Telegram  Discord  Slack  iMessage  Signal  ...       │
└────────────────────────┬────────────────────────────────────────┘
                         │ 消息收发
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Gateway（网关）                             │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  HTTP 服务器  │  │ WebSocket 服务│  │   渠道适配器          │  │
│  │ /v1/chat/..  │  │ 连接 & 事件流 │  │ (server-channels.ts) │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Agent 执行   │  │  消息路由    │  │    插件系统            │  │
│  │  (agent.ts)  │  │ (routing/)   │  │  (server-plugins.ts)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  认证 & 安全  │  │  配置热重载  │  │   定时任务 (Cron)     │  │
│  │  (auth.ts)   │  │(config-reload)│  │  (server-cron.ts)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ WebSocket
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        macOS 应用    iOS 应用    Web 控制界面
       Android 应用    CLI       自动化脚本
```

### 4.2 Gateway 连接生命周期

```
客户端                          Gateway
  │                                │
  │──── req:connect ─────────────► │  验证 token / 设备配对
  │ ◄── res(ok / error) ──────────│  返回 hello-ok（含 presence + health 快照）
  │                                │
  │ ◄── event:presence ───────────│  推送存在状态
  │ ◄── event:tick ───────────────│  心跳事件
  │                                │
  │──── req:agent ───────────────► │  触发 Agent 执行
  │ ◄── res:agent(accepted) ──────│  异步确认（含 runId）
  │ ◄── event:agent(streaming) ───│  流式响应块
  │ ◄── res:agent(final) ─────────│  最终结果
```

### 4.3 Wire 协议格式

```json
// 请求帧
{ "type": "req", "id": "abc123", "method": "agent", "params": { ... } }

// 响应帧
{ "type": "res", "id": "abc123", "ok": true, "payload": { ... } }

// 事件帧
{ "type": "event", "event": "agent", "payload": { ... }, "seq": 42 }
```

---

## 5. 源码模块详解（src/）

### 5.1 入口文件

#### `openclaw.mjs`（Bootstrap 脚本）
```
Node.js 版本检查（≥22.12）
    ↓
过滤 Bootstrap 警告
    ↓
动态加载 dist/entry.js
```

#### `src/entry.ts`（CLI 主入口）

- 检测是否为主模块（防止被 bundle 二次执行）
- 设置 `process.title = "openclaw"`
- 启用 Node.js 编译缓存
- 处理颜色标志（`--no-color`）
- 必要时 respawn 子进程（抑制实验性 API 警告）
- 快速路径：`--version`、`--help`（不加载全部模块）
- 通过 `import("./cli/run-main.js")` 懒加载完整 CLI

#### `src/index.ts`（Library 入口）
向外导出核心 API，供第三方集成使用。

---

### 5.2 CLI 层（`src/cli/`）

| 文件 | 职责 |
|------|------|
| `program.ts` | Commander.js 主程序构建器，注册所有子命令 |
| `argv.ts` | 参数解析，识别 `--version`、`--help` 等快速路径 |
| `profile.ts` | CLI Profile 环境切换 |
| `respawn-policy.ts` | 判断是否需要 respawn 子进程 |
| `daemon-cli.ts` | `daemon` 子命令实现 |
| `acp-cli.ts` | Apple Claw Protocol 子命令 |
| `progress.ts` | 终端进度条 / Spinner（基于 `@clack/prompts`） |
| `run-main.ts` | 实际调用 Commander 解析并执行 |

---

### 5.3 命令层（`src/commands/`）

实现具体的 CLI 子命令：

```
commands/
├── agent/         # agent add/bind/delete/list/identity
├── channel/       # 渠道配置与管理
├── config/        # 配置读写
├── gateway/       # gateway run/status/stop
├── onboard/       # 引导向导
├── message/       # 消息发送
├── session/       # 会话管理
├── skill/         # 技能管理
├── secrets/       # 密钥审计
└── update/        # 自动更新
```

---

### 5.4 Gateway 核心（`src/gateway/`）

这是整个项目最核心的模块，约 57,000+ 行代码，250+ 文件。

#### 主要服务文件

| 文件 | 职责 |
|------|------|
| `server.impl.ts` (~38K) | Gateway 核心逻辑，所有服务的协调器 |
| `server-http.ts` (~27K) | HTTP / WebSocket 服务器 |
| `server-chat.ts` (~20K) | 聊天协议处理 |
| `server-channels.ts` (~15K) | 渠道消息集成 |
| `server-cron.ts` (~17K) | 定时任务调度 |
| `server-node-events.ts` (~20K) | Node 事件流 |
| `server-plugins.ts` (~7K) | 插件加载与管理 |

#### 认证与安全

| 文件 | 职责 |
|------|------|
| `auth.ts` | 主认证逻辑 |
| `startup-auth.ts` | 启动时认证初始化 |
| `connection-auth.ts` | 每次连接的认证验证 |
| `device-auth.ts` | 设备配对认证 |
| `origin-check.ts` | 请求来源校验 |
| `credentials.ts` | 凭证存储与读取 |

#### HTTP API 端点

| 文件 | 职责 |
|------|------|
| `openai-http.ts` (~17K) | OpenAI 兼容 API（`/v1/chat/completions`、`/v1/models`） |
| `openresponses-http.ts` (~25K) | OpenResponses 完整协议实现 |
| `control-ui.ts` (~13K) | Web 控制界面 API 端点 |

#### 配置与状态

| 文件 | 职责 |
|------|------|
| `config-reload.ts` | 配置热重载（文件监视 + 无重启生效） |
| `server-runtime-config.ts` | 运行时配置状态 |
| `session-utils.ts` (~28K) | 会话管理工具 |
| `sessions-patch.ts` | 会话状态补丁 |

#### 客户端管理

| 文件 | 职责 |
|------|------|
| `client.ts` (~18K) | WS 连接对象，管理订阅、心跳、消息分发 |

---

### 5.5 Agent 执行层（`src/agents/`）

Agent 是 OpenClaw 的智能执行单元，负责接收用户输入、调用 LLM、执行工具并返回结果。

| 文件 | 职责 |
|------|------|
| `agent.ts` (~40K) | Agent 主协调器：LLM 调用、工具执行、结果交付 |
| `auth.ts` | Agent 认证 |
| `boot.ts` | Agent 启动初始化 |
| `call.ts` (~30K) | 语音通话 Agent 处理 |
| `canvas-capability.ts` | Canvas 渲染能力 |
| `sandbox/` | 安全沙箱（代码隔离执行） |

---

### 5.6 渠道集成（`src/channels/`）

每个消息平台都有对应的适配器，通过统一接口与 Gateway 交互。

#### 内置渠道（src/ 下独立目录）

| 目录 | 平台 |
|------|------|
| `src/discord/` | Discord |
| `src/signal/` | Signal |
| `src/imessage/` | iMessage |
| `src/slack/` | Slack |
| `src/telegram/` | Telegram |
| `src/line/` | LINE |
| `src/whatsapp/` | WhatsApp（via Baileys） |

#### 渠道插件（extensions/ 下）

| 扩展 | 平台 |
|------|------|
| `extensions/bluebubbles/` | BlueBubbles |
| `extensions/discord/` | Discord（扩展版） |
| `extensions/feishu/` | 飞书 |
| `extensions/googlechat/` | Google Chat |
| `extensions/irc/` | IRC |
| `extensions/matrix/` | Matrix |
| `extensions/mattermost/` | Mattermost |
| `extensions/msteams/` | Microsoft Teams |
| `extensions/nextcloud-talk/` | Nextcloud Talk |
| `extensions/nostr/` | Nostr |
| `extensions/synology-chat/` | 群晖 Chat |
| `extensions/telegram/` | Telegram（扩展版） |
| `extensions/tlon/` | Tlon |
| `extensions/twitch/` | Twitch |
| `extensions/whatsapp/` | WhatsApp（扩展版） |
| `extensions/zalo/` | Zalo（官方 API） |
| `extensions/zalouser/` | Zalo（个人账号） |

#### 渠道接口抽象（`src/channels/`）

```
src/channels/
├── channel-config.ts        # 渠道配置结构
├── ack-reactions.ts         # 消息已读/反应处理
├── account-snapshot-fields.ts # 账号数据快照
├── plugins/                 # 运行时渠道插件加载
└── web/                     # WebChat 界面
```

---

### 5.7 消息路由（`src/routing/`）

接收来自渠道的消息后，根据配置决定由哪个 Agent 处理：

```
inbound message
      ↓
  路由规则匹配（routing/）
      ↓
  确定目标 Agent
      ↓
  agents/ 执行
      ↓
  格式化响应
      ↓
  server-channels.ts 发回渠道
```

---

### 5.8 基础设施层（`src/infra/`，72 个文件）

提供整个项目依赖的底层工具：

| 文件 | 职责 |
|------|------|
| `env.ts` | 环境变量规范化 |
| `errors.ts` | 错误格式化与分类 |
| `ports.ts` | 端口可用性检测 |
| `binaries.ts` | 系统二进制文件（ffmpeg 等）管理 |
| `bonjour-discovery.ts` | mDNS/Bonjour 服务发现 |
| `archive.ts` | 文件归档 |
| `runtime-guard.ts` | 运行时环境校验 |
| `is-main.ts` | 模块主入口检测 |
| `warning-filter.ts` | 过滤 Node.js 警告输出 |
| `git-commit.ts` | Git commit hash 解析 |
| `openclaw-exec-env.ts` | 进程标记（防重复执行） |
| `format-time/` | 时间格式化工具（统一入口） |

---

### 5.9 媒体处理（`src/media/`）

| 文件 | 职责 |
|------|------|
| `audio.ts` | 音频处理（录制、转码） |
| `ffmpeg-exec.ts` | FFmpeg 封装执行 |
| `image-ops.ts` (~14K) | 图像操作（缩放、格式转换、裁剪） |
| `fetch.ts` | 媒体文件下载 |

---

### 5.10 LLM 提供商（`src/providers/`）

| 文件 / 目录 | 平台 |
|------------|------|
| `github-copilot/` | GitHub Copilot |
| `google-gemini/` | Google Gemini |
| `qwen/` | 通义千问（阿里云） |
| `minimax/` | MiniMax |
| 通用工具 | 共享 LLM 调用封装 |

---

### 5.11 插件系统（`src/plugins/`）

```
src/plugins/
├── discovery.ts    # 扫描并发现插件
├── loader.ts       # 动态加载插件模块
├── hooks.ts        # Hook 注册中心
├── sdk.ts          # Plugin SDK 定义
└── runtime.ts      # 运行时插件管理
```

插件生命周期：
```
1. 启动时扫描 extensions/ 目录
2. 读取各插件的 package.json
3. 动态 import 插件入口
4. 注册 Hook（消息前置/后置、自定义方法等）
5. 运行时按需调用
```

---

### 5.12 其他核心模块

| 模块 | 路径 | 职责 |
|------|------|------|
| 配置系统 | `src/config/` | 配置加载、验证、热重载 |
| 内存/知识库 | `src/memory/` | 上下文存储与检索 |
| 会话管理 | `src/sessions/` | 会话状态跟踪 |
| 安全 | `src/security/` | 密钥管理、安全审计 |
| 上下文引擎 | `src/context-engine/` | 知识检索与上下文构建 |
| 文本转语音 | `src/tts/` | TTS 集成 |
| Daemon 管理 | `src/daemon/` | 后台服务管理（launchd/systemd） |
| 定时任务 | `src/cron/` | 计划任务调度 |
| Hooks 系统 | `src/hooks/` | 生命周期事件扩展点 |
| Markdown | `src/markdown/` | Markdown 解析与格式化 |
| 国际化 | `src/i18n/` | 多语言支持 |
| 日志 | `src/logging/` | 结构化日志 |
| 终端 UI | `src/terminal/` | 表格、主题、颜色输出 |
| 浏览器自动化 | `src/browser/` | Playwright/Puppeteer 封装 |
| Web 服务 | `src/web/` | Web 服务器路由 |

---

## 6. 原生应用（apps/）

### 6.1 iOS 应用（`apps/ios/`，Swift + SwiftUI）

```
apps/ios/
├── Sources/
│   ├── Chat/           # 聊天界面
│   ├── Contacts/       # 联系人管理
│   ├── Calendar/       # 日历集成
│   ├── Reminders/      # 提醒事项
│   ├── Status/         # 状态显示
│   ├── Onboarding/     # 引导流程
│   └── Capabilities/   # 设备能力路由
├── Tests/
└── Info.plist          # 版本信息
```

状态管理：使用 **`Observation` 框架**（`@Observable`、`@Bindable`），不使用已废弃的 `ObservableObject`。

### 6.2 Android 应用（`apps/android/`，Kotlin）

```
apps/android/
├── app/               # 主应用模块
│   ├── src/
│   └── build.gradle.kts
└── benchmark/         # 性能基准测试
```

### 6.3 macOS 应用（`apps/macos/`，Swift + SwiftUI）

菜单栏应用，同时承担 Gateway 的启动/停止控制。

### 6.4 共享 Swift 框架（`apps/shared/`，OpenClawKit）

- Canvas 渲染
- 通用工具函数
- 跨 iOS/macOS 共享逻辑

---

## 7. 扩展/插件系统（extensions/）

### 7.1 扩展目录概览

```
extensions/
├── 消息渠道扩展（17个）
│   ├── bluebubbles/    ├── discord/      ├── feishu/
│   ├── googlechat/     ├── irc/          ├── line/
│   ├── matrix/         ├── mattermost/   ├── msteams/
│   ├── nextcloud-talk/ ├── nostr/        ├── signal/
│   ├── slack/          ├── synology-chat/├── telegram/
│   ├── tlon/           ├── twitch/       ├── whatsapp/
│   ├── zalo/           └── zalouser/
│
├── 认证扩展（3个）
│   ├── google-gemini-cli-auth/
│   ├── minimax-portal-auth/
│   └── qwen-portal-auth/
│
├── 内存/知识库扩展（2个）
│   ├── memory-core/
│   └── memory-lancedb/
│
├── 语音扩展（2个）
│   ├── talk-voice/
│   └── voice-call/
│
├── 其他扩展
│   ├── acpx/           # Apple Claw Protocol 扩展
│   ├── copilot-proxy/  # GitHub Copilot 代理
│   ├── device-pair/    # 设备配对
│   ├── diagnostics-otel/ # OpenTelemetry 诊断
│   ├── diffs/          # 代码 Diff 工具
│   ├── llm-task/       # LLM 任务批处理
│   ├── lobster/        # 终端 UI 工具集
│   ├── open-prose/     # 文本处理
│   ├── phone-control/  # 手机控制
│   ├── thread-ownership/ # 线程归属管理
│   └── test-utils/     # 测试工具（开发用）
│
└── shared/             # 扩展间共享代码
```

### 7.2 扩展结构规范

每个扩展是一个独立的 npm 包：

```
extensions/my-extension/
├── package.json          # 依赖在 dependencies（不用 workspace:*）
├── src/
│   ├── index.ts          # 扩展入口
│   ├── channel.ts        # 渠道实现（如有）
│   └── *.test.ts         # 测试
└── tsconfig.json
```

**依赖规则**：
- 运行时依赖放 `dependencies`（`npm install --omit=dev` 会安装）
- `openclaw` 放 `devDependencies` 或 `peerDependencies`（运行时通过 jiti alias 解析）
- 不使用 `workspace:*` 在 `dependencies` 中（生产 npm install 会报错）

---

## 8. 文档结构（docs/）

文档由 **Mintlify** 托管于 `https://docs.openclaw.ai`。

```
docs/
├── channels/         # 各渠道配置指南
├── concepts/         # 核心概念（架构、Agent、Session、模型等）
├── install/          # 安装指南
├── gateway/          # Gateway 配置与运维
├── plugins/          # 插件开发文档
├── providers/        # LLM 提供商配置
├── cli/              # CLI 参考手册
├── security/         # 安全策略
├── start/            # 入门指南
├── platforms/        # 平台特定（macOS、Linux、Windows）
├── automation/       # 自动化
├── design/           # 设计文档
├── zh-CN/            # 中文翻译（自动生成）
├── ja-JP/            # 日文翻译（自动生成）
└── docs.json         # Mintlify 导航配置
```

**中文文档**：`docs/zh-CN/` 目录下，由 i18n 流水线自动翻译，不直接手动编辑。

---

## 9. 测试基础设施

### 9.1 测试配置文件

| 配置文件 | 用途 |
|----------|------|
| `vitest.config.ts` | 主配置，并发执行 |
| `vitest.unit.config.ts` | 快速单元测试 |
| `vitest.gateway.config.ts` | Gateway 集成测试 |
| `vitest.channels.config.ts` | 渠道适配器测试 |
| `vitest.extensions.config.ts` | 扩展测试 |
| `vitest.e2e.config.ts` | 端到端测试 |
| `vitest.live.config.ts` | 真实 API 集成测试（需凭证） |
| `vitest.scoped-config.ts` | 隔离范围测试 |

### 9.2 测试组织方式

```
测试文件位置：
├── src/**/*.test.ts          # 与源码同目录的单元测试
├── src/**/*.e2e.test.ts      # 端到端测试
├── test/                     # 跨模块集成测试
├── test-fixtures/            # 测试用固定数据
└── extensions/**/src/*.test.ts # 扩展测试
```

### 9.3 测试运行命令

```bash
pnpm test                    # 运行全部测试
pnpm test:coverage           # 含覆盖率报告
pnpm test:unit               # 仅单元测试

# 低内存环境（如 CI）
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test

# 真实 API 测试（需 API Key）
CLAWDBOT_LIVE_TEST=1 pnpm test:live
```

### 9.4 测试工具（`src/test-helpers/`）

| 文件 | 职责 |
|------|------|
| `test-helpers.server.ts` | Gateway 测试 Harness |
| `test-helpers.mocks.ts` | 通用 Mock 工厂 |
| `test-helpers.e2e.ts` | E2E 测试工具 |
| `test-helpers.openai-mock.ts` | OpenAI API Mock |

---

## 10. 构建与部署

### 10.1 本地构建

```bash
pnpm install          # 安装依赖
pnpm build            # 编译到 dist/（使用 tsdown）
pnpm tsgo             # TypeScript 类型检查
pnpm check            # Lint + 格式检查（Oxlint + Oxfmt）
pnpm format:fix       # 自动修复格式
```

### 10.2 构建产物（`dist/`）

tsdown 打包的主要产物：
- `dist/entry.js` — CLI 入口
- `dist/index.js` — Library 入口
- `dist/plugin-sdk/` — Plugin SDK（35+ 导出点）
- `dist/cli/daemon-cli.js` — Daemon CLI
- 各渠道运行时模块

### 10.3 Docker 部署

多阶段构建，`Dockerfile` 三个阶段：

```
Stage 1: ext-deps
  └─ 提取扩展的 package.json

Stage 2: build
  ├─ pnpm install（含开发依赖）
  ├─ pnpm build:docker
  ├─ pnpm ui:build
  └─ pnpm prune --prod

Stage 3: runtime（基础镜像 node:22-bookworm）
  ├─ 仅复制 dist/、node_modules、docs、extensions
  ├─ 可选：Chromium（浏览器自动化）
  ├─ 可选：Docker CLI
  ├─ 非 root 用户（node:node）
  └─ 健康检查（/healthz，每 3 分钟）
```

**构建参数**：

| 参数 | 说明 |
|------|------|
| `OPENCLAW_EXTENSIONS` | 启用的扩展列表（逗号分隔） |
| `OPENCLAW_VARIANT` | `default` 或 `slim` |
| `OPENCLAW_INSTALL_BROWSER` | 是否包含 Playwright |
| `OPENCLAW_INSTALL_DOCKER_CLI` | 是否包含 Docker CLI |

**运行时环境变量**：

| 变量 | 说明 |
|------|------|
| `OPENCLAW_STATE_DIR` | 持久化数据目录（`/data`） |
| `NODE_ENV` | `production` |
| `NODE_OPTIONS` | `--max-old-space-size=1536` |

### 10.4 Fly.io 部署（`fly.toml`）

```toml
app = "openclaw"
primary_region = "iad"  # 美东

[processes]
app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

[http_service]
internal_port = 3000
force_https = true
min_machines_running = 1

[[vm]]
size = "shared-cpu-2x"
memory = "2048mb"

[[mounts]]
source = "openclaw_data"
destination = "/data"  # 持久化存储
```

---

## 11. 消息数据流

### 11.1 入站消息处理

```
用户在 WhatsApp 发送消息
         │
         ▼
Baileys WebSocket 接收原始消息
         │
         ▼
src/whatsapp/（或 extensions/whatsapp/）
  规范化为内部消息格式
         │
         ▼
src/gateway/server-channels.ts
  路由分发
         │
         ▼
src/routing/
  匹配路由规则 → 确定目标 Agent
         │
         ▼
src/agents/agent.ts
  1. 加载 Agent 配置
  2. 构建上下文（记忆 + 历史）
  3. 调用 LLM（OpenAI / Gemini / Qwen / ...）
  4. 执行工具调用（如有）
  5. 生成最终响应
         │
         ▼
src/gateway/server-channels.ts
  将响应发回 WhatsApp
         │
         ▼
用户收到 AI 回复
```

### 11.2 OpenAI 兼容 API 请求流

```
POST /v1/chat/completions
         │
         ▼
openai-http.ts
  1. 验证 Bearer token
  2. 解析请求体（messages、model、stream）
  3. 路由至目标 Agent
  4. 流式 / 非流式响应
         │
         ▼
agents/agent.ts（同上）
         │
         ▼
text/event-stream 或 JSON 响应
```

### 11.3 WebSocket 客户端连接流

```
macOS App / Web UI
         │
         ▼
WebSocket 握手（ws://127.0.0.1:18789）
         │
         ▼
server-http.ts
  建立 WS 连接
         │
         ▼
client.ts
  1. 发送 connect 帧
  2. 验证 token + 设备配对
  3. 返回 hello-ok（presence + health 快照）
  4. 订阅事件（tick、agent、presence）
         │
         ▼
持续双向通信：
  客户端 → req:* → Gateway 处理 → res:*
  Gateway → event:* → 客户端
```

---

## 12. 开发工作流

### 12.1 环境搭建

```bash
# 前提：Node.js ≥ 22、pnpm
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install

# 启动 Gateway（前台，开发模式）
pnpm openclaw gateway --port 18789 --verbose

# 或直接运行 CLI
pnpm openclaw --help
```

### 12.2 常用开发命令

```bash
pnpm build              # 编译
pnpm tsgo               # 类型检查
pnpm check              # Lint + 格式
pnpm format:fix         # 自动修正格式
pnpm test               # 运行测试
pnpm test:coverage      # 覆盖率报告
pnpm openclaw onboard   # 交互式引导向导
```

### 12.3 代码规范

- **TypeScript ESM**，严格模式，禁用 `any`
- 禁止 `@ts-nocheck` 和禁用 `no-explicit-any`
- 文件控制在 ~700 行以内（超过则拆分）
- 测试文件与源文件同目录（`*.test.ts`）
- 动态 import 与静态 import 不混用同一模块
- 不通过原型变异共享类行为（`applyPrototypeMixins` 等禁用）
- iOS/macOS 用 `Observation` 框架，不用旧的 `ObservableObject`

### 12.4 提交规范

- 遵循 Conventional Commits 风格（如 `CLI: add verbose flag to send`）
- 使用 `scripts/committer "<msg>" <file...>` 创建提交（精确控制暂存范围）
- 一个 PR 对应一个 issue，不捆绑无关改动
- PR 行数上限约 5,000 行

### 12.5 版本发布

版本号格式：`vYYYY.M.D`（稳定版）、`vYYYY.M.D-beta.N`（Beta 版）

版本信息需更新的文件：
1. `package.json`
2. `apps/android/app/build.gradle.kts`（versionName/versionCode）
3. `apps/ios/Sources/Info.plist`（CFBundleShortVersionString/CFBundleVersion）
4. `apps/macos/Sources/OpenClaw/Resources/Info.plist`

---

## 附录：关键文件速查

| 文件 | 说明 |
|------|------|
| `openclaw.mjs` | CLI 可执行文件入口（Bootstrap） |
| `src/entry.ts` | TypeScript 主入口，含 respawn 逻辑 |
| `src/index.ts` | Library 导出入口 |
| `src/cli/program.ts` | Commander 主程序构建 |
| `src/gateway/server.impl.ts` | Gateway 核心实现（最重要的文件） |
| `src/gateway/server-http.ts` | HTTP / WebSocket 服务器 |
| `src/gateway/openai-http.ts` | OpenAI 兼容 API |
| `src/agents/agent.ts` | Agent 执行协调器 |
| `src/channels/` | 渠道抽象层 |
| `src/infra/` | 底层基础设施工具 |
| `src/plugins/` | 插件系统 |
| `Dockerfile` | 多阶段容器构建 |
| `fly.toml` | Fly.io 云部署配置 |
| `package.json` | 项目依赖与脚本 |
| `tsdown.config.ts` | 构建打包配置 |
| `vitest.config.ts` | 测试配置 |
| `VISION.md` | 项目愿景与路线图 |
| `CONTRIBUTING.md` | 贡献指南 |
| `SECURITY.md` | 安全策略 |

---

*本文档基于 OpenClaw v2026.3.9 源码分析生成。*
