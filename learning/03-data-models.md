# Page Agent 数据模型

本文档整理项目中所有核心数据结构和类型定义。

---

## 一、LLM 层（`packages/llms/src/types.ts`）

### Message
```typescript
interface Message {
  role: 'system' | 'user' | 'assistant' | 'tool'
  content?: string | null
  tool_calls?: {
    id: string
    type: 'function'
    function: { name: string; arguments: string /* JSON */ }
  }[]
  tool_call_id?: string
  name?: string
}
```
> 遵循 OpenAI 标准格式，是 LLM 交互的基本单元。

### Tool
```typescript
interface Tool<TParams = any, TResult = any> {
  description?: string
  inputSchema: z.ZodType<TParams>  // Zod schema 描述参数
  execute: (args: TParams) => Promise<TResult>
}
```
> LLM-agnostic 的工具定义，通过 Zod schema 实现类型安全。

### LLMConfig
```typescript
interface LLMConfig {
  baseURL: string              // API 地址
  model: string                // 模型名称
  apiKey?: string              // API Key
  temperature?: number         // 已弃用
  maxRetries?: number          // 最大重试次数（默认 2）
  transformRequestBody?: (body: Record<string, unknown>) => Record<string, unknown> | undefined
  disableNamedToolChoice?: boolean  // 兼容不支持 tool_choice object 的模型
  customFetch?: typeof globalThis.fetch  // 自定义 fetch
}
```

### InvokeResult
```typescript
interface InvokeResult<TResult = unknown> {
  toolCall: { name: string; args: any }
  toolResult: TResult
  usage: {
    promptTokens: number
    completionTokens: number
    totalTokens: number
    cachedTokens?: number
    reasoningTokens?: number
  }
  rawResponse?: unknown
  rawRequest?: unknown
}
```

### InvokeError 分类
| 错误类型 | 含义 | 可重试 |
|----------|------|--------|
| `NETWORK_ERROR` | 网络请求失败 | ✅ |
| `AUTH_ERROR` | 认证失败（401/403） | ❌ |
| `RATE_LIMIT` | 限流（429） | ✅ |
| `SERVER_ERROR` | 服务器错误（5xx） | ✅ |
| `CONTEXT_LENGTH` | 上下文超长 | ❌ |
| `CONTENT_FILTER` | 内容过滤 | ❌ |
| `INVALID_RESPONSE` | 响应格式错误 | ❌ |
| `NO_TOOL_CALL` | 响应无工具调用 | ❌ |
| `INVALID_TOOL_ARGS` | 工具参数解析失败 | ❌ |
| `TOOL_EXECUTION_ERROR` | 工具执行失败 | ❌ |
| `CONFIG_ERROR` | 配置错误 | ❌ |

---

## 二、DOM 层（`packages/page-controller/`）

### FlatDomTree
```typescript
interface FlatDomTree {
  rootId: string
  map: Record<string, DomNode>
}
```

### DomNode 类型族
```typescript
type DomNode = TextDomNode | ElementDomNode | InteractiveElementDomNode

interface TextDomNode {
  type: 'TEXT_NODE'
  text: string
  isVisible: boolean
}

interface ElementDomNode {
  tagName: string
  attributes?: Record<string, string>
  xpath?: string
  children?: string[]          // 子节点 ID 列表
  isVisible?: boolean
  isTopElement?: boolean       // 是否为顶层元素（elementFromPoint 可见）
  isInViewport?: boolean
  isNew?: boolean              // 是否为本步新出现
  isInteractive?: false
  highlightIndex?: number      // 高亮索引（用于 LLM 操作）
  extra?: Record<string, any>  // 扩展数据（如 scrollable）
}

interface InteractiveElementDomNode extends Omit<ElementDomNode, 'isInteractive'> {
  isInteractive: true
  highlightIndex: number       // 必须有索引
  ref: HTMLElement             // 真实 DOM 引用
}
```

### BrowserState
```typescript
interface BrowserState {
  url: string
  title: string
  header: string   // 页面元信息 + 滚动位置提示
  content: string  // flatTreeToString() 结果
  footer: string   // 底部滚动提示
}
```

### DomConfig
```typescript
interface DomConfig {
  viewportExpansion?: number     // -1=全页，0=仅视口，>0=扩展像素
  interactiveBlacklist?: (Element | (() => Element))[]
  interactiveWhitelist?: (Element | (() => Element))[]
  includeAttributes?: string[]   // 额外包含的 HTML 属性
  highlightOpacity?: number
  highlightLabelOpacity?: number
  keepSemanticTags?: boolean     // 保留 nav/header/footer 等语义标签
}
```

### ActionResult（内部）
```typescript
interface ActionResult {
  success: boolean
  message: string  // 带 emoji 的人类可读结果（✅/❌）
}
```

---

## 三、Core 层（`packages/core/src/types.ts`）

### AgentConfig
```typescript
interface AgentConfig extends LLMConfig {
  language?: 'en-US' | 'zh-CN'
  maxSteps?: number                  // 默认 40
  customTools?: Record<string, PageAgentTool | null>
  instructions?: {
    system?: string
    getPageInstructions?: (url: string) => string | undefined | null
  }
  onBeforeStep?: (agent, stepCount) => Promise<void> | void
  onAfterStep?: (agent, history) => Promise<void> | void
  onBeforeTask?: (agent) => Promise<void> | void
  onAfterTask?: (agent, result) => Promise<void> | void
  onDispose?: (agent, reason?) => void
  experimentalScriptExecutionTool?: boolean  // 启用 JS 执行工具
  experimentalLlmsTxt?: boolean              // 抓取 /llms.txt
  transformPageContent?: (content: string) => Promise<string> | string
  customSystemPrompt?: string
  stepDelay?: number               // 步骤间隔（秒），默认 0.4
}
```

### AgentReflection
```typescript
interface AgentReflection {
  evaluation_previous_goal: string  // 对上一步的评估
  memory: string                    // 需要记住的信息
  next_goal: string                 // 下一步目标
}
```

### MacroToolInput
```typescript
interface MacroToolInput extends Partial<AgentReflection> {
  action: Record<string, any>  // { toolName: toolInput }
}
```

### MacroToolResult
```typescript
interface MacroToolResult {
  input: MacroToolInput
  output: string  // 工具执行结果
}
```

### HistoricalEvent（历史事件联合类型）
```typescript
type HistoricalEvent =
  | AgentStepEvent       // 步骤事件
  | ObservationEvent     // 系统观察
  | UserTakeoverEvent    // 用户接管
  | RetryEvent           // LLM 重试
  | AgentErrorEvent      // 错误

// 各子类型：
interface AgentStepEvent {
  type: 'step'
  stepIndex: number
  reflection: Partial<AgentReflection>
  action: { name: string; input: any; output: string }
  usage: { promptTokens; completionTokens; totalTokens; cachedTokens?; reasoningTokens? }
  rawResponse?: unknown
  rawRequest?: unknown
}

interface ObservationEvent {
  type: 'observation'
  content: string
}

interface UserTakeoverEvent {
  type: 'user_takeover'
}

interface RetryEvent {
  type: 'retry'
  message: string
  attempt: number
  maxAttempts: number
}

interface AgentErrorEvent {
  type: 'error'
  message: string
  rawResponse?: unknown
}
```

### AgentStatus（状态机）
```typescript
type AgentStatus = 'idle' | 'running' | 'completed' | 'error' | 'stopped'

// 状态转移：
// idle → running → completed | error | stopped
// completed/error/stopped → idle（新任务）
```

### AgentActivity（活动事件）
```typescript
type AgentActivity =
  | { type: 'thinking' }
  | { type: 'executing'; tool: string; input: unknown }
  | { type: 'executed'; tool: string; input: unknown; output: string; duration: number }
  | { type: 'retrying'; attempt: number; maxAttempts: number }
  | { type: 'error'; message: string }
```
> 瞬态，仅用于 UI 实时反馈，不进入 LLM 上下文。

### ExecutionResult
```typescript
interface ExecutionResult {
  success: boolean
  data: string               // Agent 最终回复或错误信息
  history: HistoricalEvent[] // 完整历史
}
```

---

## 四、工具层（`packages/core/src/tools/index.ts`）

### PageAgentTool
```typescript
interface PageAgentTool<TParams = any> {
  description: string
  inputSchema: z.ZodType<TParams>
  execute: (this: PageAgentCore, args: TParams, ctx: ToolContext) => Promise<string>
}

interface ToolContext {
  signal: AbortSignal  // 必须遵守，支持协作取消
}
```
> 与 LLM 层 `Tool` 的区别：`execute` 通过 `this` 绑定访问 `PageAgentCore` 实例。

### 内置工具参数

| 工具 | inputSchema |
|------|-------------|
| `done` | `{ text: string, success: boolean }` |
| `wait` | `{ seconds: number (1-10) }` |
| `ask_user` | `{ question: string }` |
| `click_element_by_index` | `{ index: int }` |
| `input_text` | `{ index: int, text: string }` |
| `select_dropdown_option` | `{ index: int, text: string }` |
| `scroll` | `{ down: boolean, num_pages: number, pixels?: int, index?: int }` |
| `scroll_horizontally` | `{ right: boolean, pixels: int, index?: int }` |
| `execute_javascript` | `{ script: string }` |

---

## 五、数据流向总览

```
用户输入 task
    │
    ▼
PageAgentCore.execute(task)
    │
    ├──→ PageController.getBrowserState()
    │       └──→ updateTree() → FlatDomTree → flatTreeToString() → BrowserState
    │
    ├──→ 组装 Prompt（system + user[ instructions + state + history + browser_state ]）
    │
    ├──→ LLM.invoke(messages, MacroTool)
    │       └──→ OpenAIClient → API → 解析 tool_call → 执行 MacroTool
    │
    ├──→ MacroTool.execute(MacroToolInput)
    │       ├── 提取 reflection → 写入 history
    │       └── 找到具体 PageAgentTool → tool.execute(toolInput)
    │               └──→ PageController.clickElement / inputText / scroll ...
    │
    ├──→ 结果写入 history → 下一步循环
    │
    └──→ 完成/失败 → ExecutionResult { success, data, history }
```
