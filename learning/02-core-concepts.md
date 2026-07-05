# Page Agent 核心概念

## 1. Re-act Agent Loop（核心循环）

这是整个项目的核心机制。每一步（step）由三个阶段组成：

```
┌─────────────────────────────────────────────┐
│  step                                        │
│  ├── observe  → 感知环境（URL/页面/历史）      │
│  ├── think    → LLM 推理（Reflection + Action）│
│  └── act      → 执行工具（DOM 操作）           │
└─────────────────────────────────────────────┘
```

循环直到：
- Agent 调用 `done` 工具（任务完成或放弃）
- 步数超过 `maxSteps`（默认 40）
- 外部调用 `stop()` 中止

---

## 2. Reflection-Before-Action（先反思再行动）

这是 Page Agent 区别于普通 Agent 的核心设计理念。

每次工具调用前，LLM 必须输出三段反思：

```json
{
  "evaluation_previous_goal": "上一步行动的效果评估",
  "memory": "需要记住的关键信息（跨步骤）",
  "next_goal": "下一步的目标",
  "action": { "tool_name": { "param": "value" } }
}
```

**实现方式**：MacroTool（宏工具）

所有工具被合并为一个 `AgentOutput` 宏工具，强制 LLM 在同一个 tool call 中先输出反思，再输出 action。

```
LLM 输出 → MacroToolInput
  ├── evaluation_previous_goal（反思上一步）
  ├── memory（记忆）
  ├── next_goal（规划下一步）
  └── action（选择一个具体工具执行）
```

---

## 3. MacroTool（宏工具模式）

将多个工具合并为一个工具，通过 Zod union schema 让 LLM 选择。

```typescript
// packages/core/src/PageAgentCore.ts#packMacroTool()
const actionSchemas = Array.from(tools.entries()).map(([name, tool]) =>
  z.object({ [name]: tool.inputSchema }).describe(tool.description)
)

const actionSchema = z.union(actionSchemas)
const macroToolSchema = z.object({
  evaluation_previous_goal: z.string().optional(),
  memory: z.string().optional(),
  next_goal: z.string().optional(),
  action: actionSchema,
})
```

使用 `toolChoiceName: 'AgentOutput'` 强制 LLM 调用该工具。

---

## 4. BrowserState（浏览器状态）

PageController 为 LLM 提供的结构化页面信息：

```typescript
interface BrowserState {
  url: string       // 当前 URL
  title: string     // 页面标题
  header: string    // 页面元信息 + 上方滚动提示
  content: string   // 简化的可交互元素文本
  footer: string    // 下方滚动提示
}
```

其中 `content` 格式如下（由 `flatTreeToString()` 生成）：
```
[0]<button aria-label=Submit>Submit />
[1]<input placeholder=Search />
[2]<a href=/docs>Documentation />
  [3]<span>Chapter 1 />
```

- `[N]` = 可交互元素索引（LLM 通过索引操作）
- `*[N]` = 本步新出现的元素
- 缩进 = 父子关系
- 无索引的文本 = 纯展示内容

---

## 5. FlatDomTree（扁平 DOM 树）

将真实 DOM 转化为扁平化的节点映射表：

```typescript
interface FlatDomTree {
  rootId: string
  map: Record<string, DomNode>  // nodeId → node
}

type DomNode =
  | TextDomNode           // 文本节点
  | ElementDomNode        // 普通元素
  | InteractiveElementDomNode  // 可交互元素（有 ref + highlightIndex）
```

**判断可交互**的标准：`button`, `a`, `input`, `select`, `textarea`, 有 `click` 事件、`tabindex` 等的元素。

---

## 6. HistoricalEvent（历史事件）

Agent 的记忆系统，所有事件追加到 `history[]`，跨步骤传入 LLM：

```typescript
type HistoricalEvent =
  | AgentStepEvent      // 一步的完整记录（反思 + 行动 + 结果）
  | ObservationEvent    // 系统观察（URL 变化、等待超时等）
  | UserTakeoverEvent   // 用户接管操作
  | RetryEvent          // LLM 重试
  | AgentErrorEvent     // 错误
```

历史事件会被组装成 `<agent_history>` XML 片段作为 Prompt 的一部分。

---

## 7. User Prompt 结构

每步发给 LLM 的用户消息由以下部分组成：

```xml
<instructions>         <!-- 可选：系统指令 + 页面指令 + llms.txt -->
<agent_state>          <!-- 用户请求 + 步骤信息 -->
<agent_history>        <!-- 历史事件流 -->
  <step_1>...</step_1>
  <sys>Page navigated to → ...</sys>
</agent_history>
<browser_state>        <!-- 当前页面信息 -->
  Current Page: [Title](URL)
  Page info: 1920x1080px viewport...
  [0]<button>... />
  [1]<input>... />
</browser_state>
```

---

## 8. Observation（系统观察）

系统自动注入的观察信息，帮助 Agent 感知环境变化：

| 触发条件 | 内容 |
|----------|------|
| URL 变化 | `Page navigated to → {url}` |
| 累积等待 ≥3s | `DO NOT wait any longer` |
| 剩余 5 步 | `Consider wrapping up` |
| 剩余 2 步 | `You must finish the task or call done immediately` |

---

## 9. SimulatorMask（模拟器遮罩）

Agent 运行时的视觉层：
- 半透明遮罩阻止用户鼠标/键盘操作
- AI 光标动画（跟随元素位置移动）
- 点击动画（涟漪效果）
- 动态适配亮色/暗色模式
- DOM 提取时临时关闭 `pointer-events` 以绕过遮罩

---

## 10. 内置工具

| 工具名 | 功能 |
|--------|------|
| `done` | 完成任务，返回结果给用户 |
| `wait` | 等待 1-10 秒（智能扣除 LLM 调用时间） |
| `ask_user` | 向用户提问并等待回答 |
| `click_element_by_index` | 按索引点击元素 |
| `input_text` | 在输入框中输入文本 |
| `select_dropdown_option` | 选择下拉选项 |
| `scroll` | 垂直滚动（支持页面级或元素级） |
| `scroll_horizontally` | 水平滚动 |
| `execute_javascript` | 执行 JS（实验性，需显式开启） |
