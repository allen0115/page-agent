# 分包结构与打包关系解析

## 一、全景概览

```
                    ┌─────────────────────────────────────┐
                    │         发布到 npm 的包              │
                    │  page-agent / @page-agent/core       │
                    │  @page-agent/llms / @page-agent/ui   │
                    │  @page-agent/page-controller          │
                    │  @page-agent/mcp                     │
                    └─────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │         私有（不发布）                │
                    │  @page-agent/ext    (Chrome 扩展)    │
                    │  @page-agent/website (官网文档)      │
                    └─────────────────────────────────────┘
```

### workspaces 拓扑顺序（`package.json`）

```json
"workspaces": [
  "packages/page-controller",   ← 最底层，无内部依赖
  "packages/ui",                ← 最底层，无内部依赖
  "packages/llms",              ← 最底层，无内部依赖
  "packages/core",              ← 依赖 llms + page-controller
  "packages/page-agent",        ← 依赖 core + llms + page-controller + ui
  "packages/mcp",               ← 独立（WebSocket + MCP SDK）
  "packages/extension",         ← 顶层消费者
  "packages/website"            ← 顶层消费者
]
```

> **npm workspaces 要求按依赖拓扑排序**，否则 `npm install` 可能找不到本地包。

---

## 二、依赖关系图

```
page-agent (npm: page-agent)
  ├── @page-agent/core
  │     ├── @page-agent/llms
  │     └── @page-agent/page-controller
  ├── @page-agent/llms (re-export)
  ├── @page-agent/page-controller (re-export)
  └── @page-agent/ui

@page-agent/ext (私有)
  ├── @page-agent/core
  ├── @page-agent/llms
  ├── @page-agent/page-controller
  └── @page-agent/ui

@page-agent/website (私有)
  └── 无内部依赖（独立 React 应用）

@page-agent/mcp (独立发布)
  └── 无内部依赖（Node.js CLI 工具）
```

---

## 三、开发态 vs 发布态：Source-First 策略

这是本项目最核心的工程化设计。

### 开发态（日常开发）

```json
// packages/core/package.json
{
  "main": "./src/PageAgentCore.ts",
  "exports": { ".": { "default": "./src/PageAgentCore.ts" } }
}
```

所有 `exports` 直接指向 **TypeScript 源码**。Vite/esbuild 在开发和构建时直接编译 `.ts` 文件，无需先 build 依赖包。

### 发布态（`npm publish`）

```json
// 发布时 publishConfig 字段被提升到顶层
{
  "main": "./dist/esm/page-agent-core.js",
  "exports": { ".": { "import": "./dist/esm/page-agent-core.js" } }
}
```

通过 `prepublishOnly` → `pre-publish.js` 自动完成切换。

### 发布流水线

```
prepublishOnly                    publish                     postpublish
     │                              │                            │
     ▼                              ▼                            ▼
pre-publish.js              npm 上传 dist/ 目录           post-publish.js
  ├── 备份 package.json.bak    （files: ["dist/"]）          ├── 恢复 package.json
  ├── publishConfig 提升到顶层                              ├── 删除 LICENSE 副本
  ├── 删除 publishConfig 字段                               └── 删除 README.md 副本
  └── 复制 LICENSE (+ README.md)
```

---

## 四、各包构建配置详解

### 4.1 `@page-agent/llms`

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/index.ts` |
| 输出 | `dist/lib/page-agent-llms.js` |
| 格式 | ES Module |
| external | `chalk`, `zod`, `zod/v4` |
| d.ts | `dist/lib/index.d.ts`（unplugin-dts 打包） |

**设计意图**：纯逻辑层，外部化 chalk 和 zod，让消费方自行解析。

### 4.2 `@page-agent/page-controller`

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/PageController.ts` |
| 输出 | `dist/lib/page-controller.js` |
| 格式 | ES Module |
| external | `@page-agent/*`, `ai-motion` |
| CSS | `vite-plugin-css-injected-by-js`（CSS 内联到 JS） |
| d.ts | `dist/lib/PageController.d.ts` |

**设计意图**：
- CSS 通过 JS 动态注入（`SimulatorMask` 的样式）
- `ai-motion` 外部化（运行时动画库，不应打包进库）

### 4.3 `@page-agent/core`

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/PageAgentCore.ts` |
| 输出 | `dist/esm/page-agent-core.js` |
| 格式 | ES Module |
| external | `chalk`, `zod`, `zod/v4`, `/^@page-agent\//` |
| CSS | `vite-plugin-css-injected-by-js` |
| d.ts | `dist/esm/PageAgentCore.d.ts` |

**设计意图**：
- 用正则 `/^@page-agent\//` 外部化所有内部包（不在 core 里打包 llms/page-controller）
- 保留 `?raw` 加载 system_prompt.md 的能力

### 4.4 `page-agent`（主包）

主包有两个构建配置：

#### ESM 版本（npm 发布）

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/PageAgent.ts` |
| 输出 | `dist/esm/page-agent.js` |
| 格式 | ES Module |
| external | 同 core |

#### IIFE 版本（CDN 直接引用）

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/demo.ts` |
| 输出 | `dist/iife/page-agent.demo.js` |
| 格式 | IIFE（`<script>` 标签直接引用） |
| external | **无**（所有依赖内联打包） |
| 环境变量 | `.env` 注入 LLM 配置 |

**设计意图**：IIFE 版本用于一行 `<script>` 引入即可使用，适合快速体验。

### 4.5 `@page-agent/ui`

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/index.ts` |
| 输出 | `dist/lib/page-agent-ui.js` |
| 格式 | ES Module |
| external | 无（UI 组件自包含） |
| CSS | `vite-plugin-css-injected-by-js` |

### 4.6 `@page-agent/ext`（私有）

| 配置项 | 值 |
|--------|-----|
| 框架 | WXT（Web Extension Toolkit）+ React |
| 开发 | `wxt`（热重载） |
| 构建 | `wxt build` |
| 打包 | `wxt zip`（生成可上传 Chrome Web Store 的 zip） |
| 发布 | 不发布 npm，只发布到 Chrome Web Store |

### 4.7 `@page-agent/mcp`（独立发布）

| 配置项 | 值 |
|--------|-----|
| 入口 | `src/index.js`（纯 JS，无需构建） |
| 发布 | `files: ["src/"]` 直接发源码 |
| bin | `page-agent-mcp` CLI 命令 |
| 依赖 | `@modelcontextprotocol/sdk`, `ws`, `zod` |

### 4.8 `@page-agent/website`（私有）

| 配置项 | 值 |
|--------|-----|
| 框架 | Vite + React + TailwindCSS |
| 开发 | `vite --host` |
| 构建 | `vite build`（静态站点） |

---

## 五、构建命令关系

```
npm run build
  └── scripts/build.js
        ├── Step 1: npm run cleanup (rm -rf packages/*/dist)
        └── Step 2: parallelTask (并行构建所有包)
              ├── @page-agent/page-controller: vite build
              ├── @page-agent/llms:            vite build
              ├── @page-agent/ui:              vite build
              ├── @page-agent/core:            vite build
              ├── page-agent:                  vite build + build:demo
              ├── @page-agent/mcp:             (无 build 脚本，跳过)
              ├── @page-agent/website:         vite build (build:website)
              └── @page-agent/ext:             wxt zip

npm run build:libs
  └── scripts/build-libs.js
        └── 只构建有 build 脚本的包（不含 website 和 ext 的 zip）

npm run build:ext
  └── cd packages/extension && wxt zip

npm run build:website
  └── cd packages/website && vite build
```

---

## 六、CI 流水线

```
scripts/ci.js
  ├── 1. commitlint（校验提交信息，main 分支跳过）
  ├── 2. 并行执行：
  │     ├── npm run lint
  │     ├── npx prettier --check .
  │     ├── npm run typecheck
  │     └── npm test
  └── 3. npm run build（全量构建）
```

---

## 七、版本同步机制

```
scripts/sync-version.js [newVersion]
  ├── 更新 root package.json version
  ├── 遍历 packages/*/package.json
  │     ├── 更新 version 字段
  │     └── 更新 dependencies 中内部包的版本号
  └── 更新 CDN URL（README.md、官网常量文件）
```

所有包共享同一版本号（`1.11.0`），通过 `sync-version.js` 保证一致。

---

## 八、CSS 处理策略

所有包含 CSS 的包（page-controller、core、ui、page-agent）都使用 `vite-plugin-css-injected-by-js`：

```
构建时：CSS 被提取为独立文件
运行时：JS 加载时动态创建 <style> 标签注入 CSS
```

好处：
- 消费方无需单独引入 CSS 文件
- `<script>` 一行引入即可使用（IIFE 场景）
- 避免 CSS 文件路径解析问题

---

## 九、External 策略总结

| 包 | external |
|----|----------|
| llms | `chalk`, `zod`, `zod/v4` |
| page-controller | `@page-agent/*`, `ai-motion` |
| core | `chalk`, `zod`, `zod/v4`, `/^@page-agent\//` |
| page-agent (ESM) | `chalk`, `zod`, `zod/v4`, `/^@page-agent\//` |
| page-agent (IIFE) | **无**（全部内联） |
| ui | 无 |
| ext | WXT 自行处理 |
| website | Vite 常规处理 |

**原则**：
- **库模式**（ESM）：内部包 + 公共运行时依赖（zod/chalk）外部化，让消费方 tree-shake
- **应用模式**（IIFE/Extension）：全部内联，生成自包含产物

---

## 十、总结

```
┌─ 开发态 ─────────────────────────────────────────────────────┐
│  所有 exports 指向 src/*.ts                                   │
│  Vite/esbuild 直接编译源码，无需预构建依赖                       │
│  npm start → 启动 website 开发服务器                           │
│  npm run dev:demo → IIFE 热重载 + 本地服务器                    │
└──────────────────────────────────────────────────────────────┘

┌─ 构建态 ─────────────────────────────────────────────────────┐
│  npm run build → cleanup + 并行构建所有包                      │
│  库包 → dist/lib/*.js + dist/lib/*.d.ts                       │
│  核心包 → dist/esm/*.js + dist/esm/*.d.ts                     │
│  入口包 → dist/esm/*.js + dist/iife/*.demo.js                 │
│  扩展 → .output/ + .zip                                       │
│  网站 → dist/（静态站点）                                      │
└──────────────────────────────────────────────────────────────┘

┌─ 发布态 ─────────────────────────────────────────────────────┐
│  prepublishOnly → pre-publish.js 改写 package.json           │
│  npm publish → 上传 files: ["dist/"] 到 registry              │
│  postpublish → post-publish.js 恢复 package.json             │
│  sync-version.js → 统一版本号 + 更新 CDN URL                  │
└──────────────────────────────────────────────────────────────┘
```
