# Agent Note: 流式增量中为 null 的工具名不得覆盖起始增量中的工具名

Status: implemented

[English](2026-08-19-null-tool-call-name-clobber.md) | 中文

## Problem

一个 agent（智能体）会话在调用某个工具时可能收到 `UNKNOWN_TOOL` 且工具名为空，即使该工具（例如 `bash`）已注册、并且模型确实输出了它。UI 会话记录显示模型坚持声称自己写出了工具名，然而每次调用仍失败，看起来像是部署或预设的问题。

## Decision

`packages/llm/llm-deepseek/src/translate.ts` 对工具调用的身份判空使用 `!== undefined`：

```ts
export function preFix() {
  const call: { id?: string | null; function?: { name?: string | null } } = { id: 'c', function: { name: 'bash' } }
  const block: { callId?: string | null; name?: string | null } = {}
  if (call.id !== undefined) block.callId = call.id
  if (call.function?.name !== undefined) block.name = call.function.name
}
```

某些提供方会把一次工具调用流式输出为：起始增量携带 `function.name` 与 `id`，后续增量只携带 `arguments`，且身份字段以显式 `null`（而非缺省）出现。由于 `null !== undefined` 为 `true`，这样的后续增量会用 `null` 覆盖已打开块的合法工具名，最终工具调用块以 `name: ''` 收尾——派发时即报 `unknown tool ""`，对外表现为 `UNKNOWN_TOOL`。工具名已经注册、模型也已输出，是流式解析把它剥掉了。

现在守卫只在字段确为有效值时才覆盖：

```ts
export function postFix() {
  const call: { id?: string | null; function?: { name?: string | null } } = { id: null, function: { name: null } }
  const block: { callId?: string | null; name?: string | null } = {}
  if (call.id != null) block.callId = call.id
  if (call.function?.name != null) block.name = call.function.name
}
```

携带 `null` 名/id 的后续增量不再覆盖起始增量的值，而完全省略这些字段的增量（既有的宽松 wire 场景）行为与之前完全一致。

## Testing

`packages/llm/llm-deepseek/tests/translate.spec.ts` 新增回归测试，喂入实际的 provider 形态——起始增量带 `function.name`，随后是 `name: null`、`id: null` 的后续增量——并断言最终块保留起始增量的名称与调用 id。该测试在修复前的 `!== undefined` 守卫下失败，修复后通过。

## Alternatives considered

**在派发处拒绝空工具名。** 已否决：名称在发出时是有效的，在执行业务层拒绝只会掩盖究竟是哪个组件丢弃了它，让问题更难定位。

**当始终收不到名称时回退到 `id`。** 已否决，因为目前没有任何 wire 证据表明存在只输出 `arguments`、却从不发出工具名的 provider；`!= null` 守卫修复了观察到的 null 覆盖问题，而不需要凭空发明新的回退契约。

## Consequences

身份在起始增量与后续增量之间拆分的流式工具调用，在整个流中都能保留其名称与调用 id，使模型输出的工具名原样抵达派发。改动仅局限于流式组装器；序列化、工具注册与派发均未受影响。
