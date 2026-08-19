# Agent Note: A null tool-call name in a streaming delta must not clobber the opening name

Status: implemented

English | [中文](2026-08-19-null-tool-call-name-clobber.zh.md)

## Problem

An agent session could call a tool and receive `UNKNOWN_TOOL` with an empty tool name, even though the registered tool (for example `bash`) was present and the model emitted it. The UI transcript showed the model insisting it had written the tool name while every call still failed, which read as a deployment or preset problem.

## Decision

`packages/llm/llm-deepseek/src/translate.ts` guarded tool-call identity with `!== undefined`:

```ts
if (call.id !== undefined) block.callId = call.id
if (call.function?.name !== undefined) block.name = call.function.name
```

Some providers stream a tool call as an opening delta that carries `function.name` and `id`, followed by continuation deltas that carry only `arguments` with the identity fields present as explicit `null` (not absent). Because `null !== undefined` is `true`, such a continuation delta overwrote an already-open block's valid name with `null`, and the emitted tool-call block then closed with `name: ''` — dispatched as `unknown tool ""`, surfaced as `UNKNOWN_TOOL`. The name was registered and the model had emitted it; the streaming parser stripped it.

The guards now treat only genuinely-present values as overrides:

```ts
if (call.id != null) block.callId = call.id
if (call.function?.name != null) block.name = call.function.name
```

A continuation delta with `null` name/id no longer clobbers the opening delta's value, while deltas that omit the fields entirely (the pre-existing lenient-wire case) behave exactly as before.

## Testing

`packages/llm/llm-deepseek/tests/translate.spec.ts` adds a regression test that feeds the live provider shape — an opening delta with `function.name`, then a continuation delta with `name: null` and `id: null` — and asserts the closing block retains the opening name and call id. The test fails against the pre-fix `!== undefined` guards and passes after.

## Alternatives considered

**Reject empty-name tool calls at dispatch.** Rejected: the name was valid at emission, and rejecting in the executive would only mask which component dropped it, making the failure harder to trace.

**Also fall back to the `id` when no name arrives.** Rejected because there is no current wire evidence for a provider that streams `arguments` without ever emitting the name; the `!= null` guard fixes the observed null-clobber without inventing a new fallback contract.

## Consequences

A streaming tool call whose identity is split across opening and continuation deltas keeps its name and call id for the whole stream, so model-emitted tool names reach dispatch unchanged. The change is confined to the streaming assembler; serialization, tool registration, and dispatch are untouched.
