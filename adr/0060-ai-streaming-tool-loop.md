# ADR-0060: AI: Streaming Auto-Execute Tool Loop

**Date:** 2026-11-05
**Status:** Accepted

## Context

`AbstractClient::chat()` has an auto-execute tool loop: it sends a request,
and while the model keeps returning tool calls it executes them, appends the
results, and sends the next request — bounded by
`ChatOption::MAX_TOOL_ITERATIONS` (default 10). `AbstractClient::streamChat()`
has **no such loop**. It builds one streaming request, calls `doStreamChat()`,
and finishes. If the streamed turn ends in tool calls, nothing executes them and
the stream simply ends — the caller gets tool calls, never a final answer.

This blocks ADR-0059 (final synthesis pass on iteration-cap) for streaming:
there is no streaming loop, hence no iteration cap, hence nothing to synthesize
from. More fundamentally, **streaming + tool calling is not actually supported**
as an end-to-end capability today, even though streaming and tool calling each
work in isolation.

A provider audit (2026-11-05) shows streamed tool-call support is **inconsistent**:

| Provider / path | Surfaces tool calls from the stream? |
|---|---|
| OpenAI (`OpenAIFormatter::executeStreamChat`) | **No** — only `delta.content` is parsed |
| Anthropic (`AnthropicFormatter::executeStreamChat`) | **Yes** — `tool_use` + `input_json_delta` |
| Google Interactions API (gemini-3.x) | **Yes** — `function_call` → `ToolCall`, sets `finish_reason=tool_calls` |
| Google generateContent (gemini-2.x) | **No** — tool-call parts dropped |
| Bedrock Converse | **Yes** — `toolUse` blocks parsed |

Two of five paths drop streamed tool calls. A streaming tool loop cannot be
built uniformly until those parsers surface tool calls, and the loop must fail
cleanly on paths that cannot.

No existing ADR defines the auto-execute tool loop — it is emergent code.
ADR-0056 governs re-applying the context strategy *between* iterations of the
non-streaming loop; ADR-0059 defines the final-synthesis pass that *consumes*
the loop. Neither defines the loop or addresses streaming. This is therefore a
new decision, not an amendment.

## Decision

Introduce a **streaming auto-execute tool loop** in `AbstractClient::streamChat()`,
mirroring the non-streaming loop, and make streaming tool-call support a
first-class, provider-reported capability.

### 1. The loop

When tools are provided and auto-execute is on, `streamChat()` runs:

1. Stream a turn via `doStreamChat()`, accumulating streamed text **and** any
   tool calls the provider surfaces, plus the turn's `finish_reason`.
2. If the turn ended with tool calls, execute them with the existing
   `executeTools()` (shared with the non-streaming loop), append the assistant
   tool-call message and tool results to the conversation, and stream the next
   turn.
3. Stop when a turn ends without tool calls, or when
   `ChatOption::MAX_TOOL_ITERATIONS` is reached.

The loop reuses `executeTools()`, tool-status events, and the per-iteration
cancellation checks from the non-streaming path.

### 2. Caller-facing token semantics

**Only the final answer's tokens are delivered to `onToken`.** Intermediate
turns that end in tool calls are consumed internally (their text, if any, is not
streamed to `onToken`), because that text is a precursor to tool use, not the
answer. `onComplete` receives the final `ChatResponse`. Tool execution is still
observable through the existing tool-status events
(`TOOL_CALLING` / `TOOL_EXECUTING` / `TOOL_COMPLETED` / `TOOL_FAILED`), so a UI
can show "calling tool…" between streamed answer segments.

(Alternative — stream every turn's tokens — is rejected below.)

### 3. Provider capability contract

Surfacing tool calls from a stream is made an explicit capability. A provider
stream handler either reports streamed tool calls (via the `ChatResponse` passed
to the internal completion callback carrying tool calls, as Anthropic / Google
Interactions / Bedrock Converse already do) or it does not. When auto-execute
with tools is requested on a streaming path that **cannot** surface tool calls
(OpenAI, Google generateContent today), the library throws a clear
`UnsupportedFeatureException` rather than silently never looping.

Bringing OpenAI and Google generateContent up to parity (parsing
`tool_calls` / `functionCall` deltas in their stream handlers) is **in scope**
for this work so the capability is uniform across the mainstream providers.

### 4. Final synthesis on cap (ADR-0059) for streaming

Once the streaming loop exists, ADR-0059's trigger applies: if the loop exits on
`MAX_TOOL_ITERATIONS` with tool calls still pending and
`FINAL_SYNTHESIS_ON_MAX_ITERATIONS` is set, issue one final tools-off streaming
request whose tokens flow through `onToken` / `onComplete` as the answer. The
same Option-A handling (drop the dangling tool-call message) applies.

### 5. Cross-cutting behavior

- **Cancellation (ADR-0057):** checked between iterations and carried on each
  streamed request; a mid-loop cancel surfaces the partial via
  `ChatOption::ON_CANCEL` as today.
- **Context window (ADR-0056):** re-applied between streamed iterations under the
  same opt-in as the non-streaming loop.
- **Metrics/audit:** the existing `stream.*` metrics describe the whole turn;
  per-iteration tool metrics reuse the non-streaming events.

## Alternatives Considered

- **Stream every turn's tokens (not just the final answer) — rejected for v1.**
  Surfaces intermediate "thinking/precursor" text that is confusing as answer
  content and hard for callers to segment. Tool-status events already make tool
  use observable. Could be offered later as an opt-in verbose mode.
- **Leave streaming without tools; document the limitation — rejected.**
  Streaming + tools is a core expectation; the gap is a real defect, not a
  documentation matter.
- **Silently skip the loop on providers that cannot surface streamed tool calls
  — rejected.** That reproduces the current silent failure. Throwing
  `UnsupportedFeatureException` is honest and consistent with how other
  provider-specific gaps are handled.
- **Fold this into ADR-0059 — rejected.** 0059 consumes the loop; it cannot also
  define it without inverting the dependency. 0059 gets a cross-reference
  instead.

## Consequences

- Streaming gains true tool calling: execute-and-continue with a final streamed
  answer, bounded by the same iteration cap as non-streaming.
- Unblocks the streaming half of ADR-0059.
- Requires new stream-parsing work for OpenAI and Google generateContent, and a
  capability signal so unsupported paths fail cleanly.
- Larger blast radius than a localized change: touches `streamChat()`, every
  provider's stream handler, and the formatter/strategy contract. Must be
  covered by per-provider streaming-tool tests plus a cross-provider guard.
- New caller-visible semantics (only final-answer tokens stream); documented so
  integrators relying on current single-turn streaming are not surprised.

## Tracking

WebFiori/ai streaming-tool-loop epic (see issue). ADR-0059 (final synthesis)
depends on this for its streaming path.
