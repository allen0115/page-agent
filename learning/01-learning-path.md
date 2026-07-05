# Page Agent 学习路径

## 阶段一：基础理解

### 1. 项目定位
Page Agent 是一个**浏览器内 AI Agent**，用户输入自然语言需求，Agent 通过 LLM 理解页面内容并自动执行 DOM 操作（点击、输入、滚动等）。

### 2. Monorepo 结构（按依赖顺序）

```
packages/
├── page-controller/   # 底层：DOM 操作，无 LLM 依赖
├── llms/              # 底层：LLM 客户端，OpenAI 协议
├── ui/                # 底层：Panel UI + i18n，与 Agent 解耦
├── core/              # 中层：Agent 核心逻辑（无 UI）
├── page-agent/        # 顶层：入口包，继承 Core + UI + Controller
├── mcp/               # 顶层：MCP 协议支持
├── extension/         # 顶层：Chrome 扩展（WXT + React）
└── website/           # 顶层：官网 + 文档
```

> **学习顺序建议**：从底层往上读，因为上层依赖下层。

---

## 阶段二：核心模块深入

### Step 1：理解 LLMs 包（`packages/llms/`）

**目标**：理解 LLM 调用是怎么封装的。

| 文件 | 要点 |
|------|------|
| `src/types.ts` | `LLMConfig`、`Tool`、`Message`、`InvokeResult` 类型定义 |
| `src/OpenAIClient.ts` | 标准 OpenAI 协议：请求构造 → API 调用 → 响应解析 → 工具执行 |
| `src/index.ts` | `LLM` 类：封装重试逻辑，事件派发（retry） |
| `src/errors.ts` | `InvokeError` 错误分类（网络/认证/限流/上下文长度等） |

**关键概念**：
- `Tool<TParams, TResult>`：用 Zod schema 描述输入，LLM-agnostic
- `invoke()` 一次调用 = 一次 API 请求 + 一次工具执行
- 重试机制：区分可重试 vs 不可重试错误

---

### Step 2：理解 Page Controller（`packages/page-controller/`）

**目标**：理解 DOM 如何被提取、简化和操作。

| 文件 | 要点 |
|------|------|
| `src/PageController.ts` | 主类：管理 FlatDomTree + selectorMap + 元素操作 |
| `src/dom/dom_tree/type.ts` | `FlatDomTree`、`DomNode`（文本/元素/可交互元素）类型 |
| `src/dom/index.ts` | `getFlatTree()` → `flatTreeToString()` → `getSelectorMap()` 管线 |
| `src/actions.ts` | 实际 DOM 操作：click、input、scroll、select |
| `src/mask/SimulatorMask.ts` | 视觉遮罩层：阻止用户操作 + AI 光标动画 |

**核心管线**：
```
Live DOM → getFlatTree() → FlatDomTree
         → flatTreeToString() → 简化文本（给 LLM）
         → getSelectorMap() → Map<index, element>（给操作）
```

---

### Step 3：理解 Core 包（`packages/core/`）

**目标**：理解 Agent 循环（Re-act Loop）。

| 文件 | 要点 |
|------|------|
| `src/PageAgentCore.ts` | 核心 Agent 类，662 行，最重要的文件 |
| `src/types.ts` | 所有类型：AgentConfig、HistoricalEvent、AgentStatus 等 |
| `src/tools/index.ts` | 内置工具：done、wait、click、input、scroll 等 |
| `src/prompts/system_prompt.md` | 系统 Prompt：定义 Agent 行为规范 |

---

### Step 4：理解 Page Agent 入口（`packages/page-agent/`）

**目标**：理解最终组装。

`PageAgent` 仅 30 行，做三件事：
1. 创建 `PageController` 实例
2. 调用 `super()` 初始化 `PageAgentCore`
3. 创建 `Panel` UI 实例

---

## 阶段三：进阶理解

### 事件系统
```
PageAgentCore extends EventTarget
  ├── 'statuschange'  → 状态切换（idle → running → completed/error/stopped）
  ├── 'historychange' → 历史事件更新（持久化，进入 LLM 上下文）
  ├── 'activity'      → 实时活动反馈（瞬态，仅 UI）
  └── 'dispose'       → Agent 销毁
```

### 信息双流

| 流 | 名称 | 持久性 | 进入 LLM | 用途 |
|----|------|--------|----------|------|
| 历史事件 | `history[]` | 持久 | ✅ | Agent 记忆 |
| 活动事件 | `activity` | 瞬态 | ❌ | UI 反馈 |

### 生命周期钩子
```
onBeforeTask → [onBeforeStep → step → onAfterStep] × N → onAfterTask → onDispose
```

### 自定义扩展点
- `customTools`：扩展/覆盖/删除内置工具
- `instructions.system`：全局指令
- `instructions.getPageInstructions(url)`：按页面动态注入指令
- `transformPageContent`：内容脱敏/变换
- `customSystemPrompt`：完全覆盖系统 Prompt
- `customFetch`：自定义 LLM 请求（代理/认证等）
