# MacroTool（宏工具）深入浅出

## 一、为什么需要 MacroTool？

普通 AI Agent 框架（如 LangChain）通常把多个工具平铺给 LLM：

```
LLM 看到 N 个工具 → 选择一个调用 → 只有 action，没有思考过程
```

这带来两个问题：
1. **缺少推理过程**：LLM 直接行动，不反思上一步、不记忆、不规划
2. **多工具调用风险**：`parallel_tool_calls` 可能一次调用多个工具，导致并发副作用

Page Agent 的解法是：**把所有工具打包成一个宏工具，并在 schema 层面强制 LLM 先反思再行动**。

---

## 二、MacroTool 的构建过程

核心方法是 `#packMacroTool()`（`packages/core/src/PageAgentCore.ts` L386-L470），分三步：

### 第一步：把每个工具包成 `{ toolName: inputSchema }` 的对象

```typescript
const actionSchemas = Array.from(tools.entries()).map(([toolName, tool]) =>
  z.object({ [toolName]: tool.inputSchema }).describe(tool.description)
)
```

以 `click_element_by_index` 为例：
```
原始:   z.object({ index: z.int().min(0) })
包装后: z.object({ click_element_by_index: z.object({ index: z.int().min(0) }) })
```

### 第二步：用 `z.union` 把所有工具合并为 action 字段

```typescript
const actionSchema = z.union(actionSchemas)
// 含义：action 必须是其中一个工具的包装对象
```

### 第三步：加上反思字段，形成最终的 MacroTool schema

```typescript
const macroToolSchema = z.object({
  evaluation_previous_goal: z.string().optional(),  // 反思上一步
  memory: z.string().optional(),                    // 记忆
  next_goal: z.string().optional(),                 // 下一步规划
  action: actionSchema,                             // 选择一个工具执行
})
```

最终生成的 Zod schema 被转换为 OpenAI tool 格式发给 LLM。

---

## 三、LLM 看到的是什么？

LLM 通过 `zodToOpenAITool()`（`packages/llms/src/utils.ts` L15-L24）看到的是：

```json
{
  "type": "function",
  "function": {
    "name": "AgentOutput",
    "description": "You MUST call this tool every step!",
    "parameters": {
      "type": "object",
      "required": ["action"],
      "properties": {
        "evaluation_previous_goal": { "type": "string" },
        "memory": { "type": "string" },
        "next_goal": { "type": "string" },
        "action": {
          "oneOf": [
            { "type": "object", "properties": { "done": { ... } } },
            { "type": "object", "properties": { "wait": { ... } } },
            { "type": "object", "properties": { "click_element_by_index": { ... } } }
          ]
        }
      }
    }
  }
}
```

并且通过 `tool_choice: { type: 'function', function: { name: 'AgentOutput' } }` **强制** LLM 必须调用这个工具。

---

## 四、一次完整的 LLM 输出

```json
{
  "evaluation_previous_goal": "Successfully navigated to the product page. Verdict: Success",
  "memory": "Found product 'Wireless Headphones' at $29.99. Need to add to cart next.",
  "next_goal": "Click the 'Add to Cart' button to proceed.",
  "action": {
    "click_element_by_index": { "index": 17 }
  }
}
```

LLM **必须在同一个 tool call 里**完成：反思 + 记忆 + 规划 + 行动。这就是 Reflection-Before-Action 的强制落地。

---

## 五、MacroTool 执行时做了什么？

当 LLM 返回 `AgentOutput` tool call 后，`OpenAIClient` 解析 JSON 参数 → 用 Zod schema 验证 → 调用 `macroTool.execute(input)`：

```
macroTool.execute(MacroToolInput)
  │
  ├── 1. 提取 action: Object.keys(action)[0] 得到 toolName
  │
  ├── 2. 查找工具: tools.get(toolName)
  │
  ├── 3. 发射 activity 事件（UI 反馈 "executing..."）
  │
  ├── 4. 执行工具: tool.execute.bind(this)(toolInput, { signal })
  │      注意 this 绑定到 PageAgentCore 实例
  │      工具内部通过 this.pageController.xxx() 操作 DOM
  │
  ├── 5. 强制检查 abort（防止工具忽略信号后仍正常 resolve）
  │
  ├── 6. 发射 activity 事件（UI 反馈 "executed"）
  │
  ├── 7. 累计等待时间（用于观察告警）
  │
  └── 8. 返回 { input: MacroToolInput, output: string }
```

关键一行（L440）：
```typescript
const result = await tool.execute.bind(this)(toolInput, { signal })
```
`bind(this)` 让工具函数的 `this` 指向 `PageAgentCore`，从而能访问 `this.pageController`。

---

## 六、normalizeResponse：修复 LLM 的"乱回答"

不同模型的输出格式千奇百怪，`normalizeResponse()`（`packages/core/src/utils/autoFixer.ts` L21-L121）负责在解析前自动修复：

| 异常情况 | 修复策略 |
|----------|----------|
| 没有 `tool_calls`，JSON 在 `content` 字段里 | 从 content 中提取 JSON |
| 工具名不是 `AgentOutput` 而是具体工具名 | 包装为 `{ action: ... }` |
| arguments 是双重 JSON 字符串 | 递归 `safeJsonParse` |
| 只有 action 层，没有反思字段 | 补包 `{ action: ... }` |
| 有 2 层 function 嵌套 | 逐层拆解 |
| 缺少 action 字段 | 降级为 `{ wait: { seconds: 1 } }` |
| 单字段工具的原始值输入（如 `{"click_element_by_index": 2}`） | 自动补为 `{"click_element_by_index": {"index": 2}}` |

这是工程上非常实用的**防御性解析**模式。

### normalizeResponse 源码结构

```typescript
function normalizeResponse(response, tools) {
  // 1. 定位 arguments：优先 tool_calls，fallback content 里的 JSON
  // 2. 处理工具名不是 AgentOutput 的情况
  // 3. 递归 safeJsonParse 解包双重字符串
  // 4. validateAction() 校验 + 自动补全单字段工具
  // 5. 缺少 action 时降级为 wait
  // 6. 重新包装为标准 AgentOutput tool_call 格式
}

function validateAction(action, tools) {
  // 1. 从 action 中取出 toolName（第一个 key）
  // 2. 在 tools Map 中查找，找不到则抛出可用工具列表
  // 3. 如果 schema 是 ZodObject 但输入是原始值，自动补全字段名
  // 4. 用 schema.safeParse() 校验并返回 coerced 数据
}
```

---

## 七、MacroTool 结果如何流转到历史

回到 `execute()` 主循环（L290-L315）：

```typescript
const macroResult = result.toolResult as MacroToolResult
const input = macroResult.input           // LLM 的原始输出（含反思）
const output = macroResult.output         // 工具执行结果

// 分离反思和行动
const reflection = { evaluation_previous_goal, memory, next_goal }
const action = { name: actionName, input: toolInput, output }

// 写入历史
this.#emitHistoryChange({
  type: 'step',
  stepIndex: step,
  reflection,
  action,
  usage: result.usage,
})
```

下一步的 Prompt 中，这段历史会被格式化为：

```xml
<step_1>
Evaluation of Previous Step: Successfully navigated...
Memory: Found product at $29.99...
Next Goal: Click the Add to Cart button...
Action Results: ✅ Clicked element (Add to Cart).
</step_1>
```

**这就形成了闭环**：上一步的反思 + 结果 → 下一步 LLM 的上下文 → 指导新的反思和行动。

---

## 八、完整数据流图

```
┌─ PageAgentCore.execute() ─────────────────────────────────────────────────┐
│                                                                            │
│  ┌─ observe ────────────────────────────────────────────────────────────┐ │
│  │  PageController.getBrowserState() → BrowserState                     │ │
│  │  handleObservations() → push system observations to history          │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌─ think ──────────────────────────────────────────────────────────────┐ │
│  │  组装 messages = [system_prompt, user_prompt]                        │ │
│  │  macroTool = { AgentOutput: packMacroTool() }                        │ │
│  │                                                                       │ │
│  │  LLM.invoke(messages, macroTool)                                     │ │
│  │    → OpenAIClient → API → response                                   │ │
│  │    → normalizeResponse() (修复格式)                                   │ │
│  │    → Zod schema validate                                             │ │
│  │    → macroTool.execute(input)                                        │ │
│  │         ├── 解析 reflection（评估/记忆/规划）                          │ │
│  │         └── 执行具体工具（click/input/scroll/done...）                 │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌─ act ────────────────────────────────────────────────────────────────┐ │
│  │  从 MacroToolResult 中提取 reflection + action                       │ │
│  │  写入 history → emit historychange                                   │ │
│  │  如果 action == 'done' → break 循环                                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  step++ → 继续循环                                                        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、设计总结

| 设计点 | 效果 |
|--------|------|
| 所有工具合并为一个 MacroTool | LLM 只看到一个工具，不会乱选 |
| `tool_choice` 强制指定 AgentOutput | 确保每步都调用，不会空转 |
| 反思字段在 schema 里 | LLM **被迫**先思考再行动 |
| action 用 `z.union` 约束 | 只能选一个工具，杜绝并行 |
| `normalizeResponse` 兜底 | 兼容各种模型的格式问题 |
| `bind(this)` 注入上下文 | 工具函数能访问 `pageController` |
| 反射结果写入历史 → 下一步 Prompt | 形成记忆闭环 |

---

## 十、与其他框架的对比

| 特性 | Page Agent (MacroTool) | LangChain | OpenAI Assistants |
|------|----------------------|-----------|-------------------|
| 工具数量 | 1 个宏工具 | N 个平铺 | N 个平铺 |
| 强制反思 | schema 层面强制 | 需手动实现 | 无 |
| 并行工具调用 | 不可能（单工具） | 可能 | 可能 |
| 输出格式修复 | normalizeResponse 多层兜底 | 无 | 无 |
| 工具选择方式 | `tool_choice` 强制 | 模型自选 | 模型自选 |
