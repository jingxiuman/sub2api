# DeepSeek Thinking Continuity Design

## Goal

Fix DeepSeek Anthropic-compatible passthrough failures where DeepSeek returns:

`The content[].thinking in the thinking mode must be passed back to the API.`

The issue occurs on the Anthropic API Key passthrough path when a thinking-mode tool-call continuation reaches DeepSeek without the assistant `thinking` block that belongs to the preceding `tool_use`.

## Success Criteria

- Existing Anthropic API Key passthrough behavior remains unchanged by default.
- Operators can enable a DeepSeek-only account option without affecting normal Anthropic, Bedrock, OpenAI, Gemini, or Antigravity paths.
- `diagnose` mode detects likely missing-thinking continuation requests and logs safe structural diagnostics without changing the request body.
- `restore` mode repairs only tool-result continuation requests when a matching cached assistant `thinking + tool_use` entry exists.
- No thinking text is written to logs, persisted to the database, or exposed in ops details.
- Automated tests cover detection, no-op behavior, restoration, session isolation, and response capture.

## Non-Goals

- Do not make DeepSeek a new platform constant in this change.
- Do not change default passthrough behavior.
- Do not implement a generic Anthropic thinking repair layer.
- Do not fix clients that omit entire conversation history outside the narrow tool-result continuation case.
- Do not add Redis-backed storage in the first implementation. Multi-instance support can be added later if needed.

## Account Configuration

Use an account-level extra field:

```json
{
  "anthropic_passthrough": true,
  "deepseek_thinking_continuity_mode": "off"
}
```

Supported values:

- `off`: disabled. This is the default and preserves current behavior.
- `diagnose`: inspect requests and log safe diagnostics, but do not modify request bodies.
- `restore`: inspect requests and restore cached thinking blocks for matching tool-result continuations.

The mode is effective only when all conditions are true:

- account platform is `anthropic`
- account type is `apikey`
- `extra.anthropic_passthrough` is `true`
- account base URL is a DeepSeek Anthropic endpoint, such as `api.deepseek.com/anthropic`
- mode is `diagnose` or `restore`

## Backend Shape

Add account helpers near the existing passthrough helpers in `backend/internal/service/account.go`:

```go
const (
    DeepSeekThinkingContinuityOff      = "off"
    DeepSeekThinkingContinuityDiagnose = "diagnose"
    DeepSeekThinkingContinuityRestore  = "restore"
)

func (a *Account) GetDeepSeekThinkingContinuityMode() string
func (a *Account) IsDeepSeekThinkingContinuityEnabled() bool
```

Add a focused implementation file:

`backend/internal/service/deepseek_thinking_continuity.go`

This file owns:

- DeepSeek base URL detection
- request diagnostics
- session key derivation
- in-memory TTL cache
- non-streaming response capture
- streaming SSE response capture
- request restoration

## Data Model

Use process-local memory for the first implementation.

```go
type DeepSeekThinkingEntry struct {
    AccountID int64
    SessionKey string
    ToolUseID string
    AssistantContent []map[string]any
    CreatedAt time.Time
    ExpiresAt time.Time
}
```

The stored assistant content must contain the original `thinking` block and matching `tool_use` block. The cache stores thinking text in memory only. TTL defaults to 10 minutes.

Use a small interface to keep future Redis support isolated:

```go
type DeepSeekThinkingContinuityStore interface {
    Get(sessionKey string, toolUseID string) (*DeepSeekThinkingEntry, bool)
    Put(entry DeepSeekThinkingEntry)
    DeleteExpired(now time.Time)
}
```

## Session Isolation

Cache keys must include enough scope to avoid cross-user or cross-session leakage:

```text
deepseek-thinking:{api_key_id}:{group_id}:{account_id}:{session_signal}
```

Session signal priority:

1. request header `session_id`
2. request header `conversation_id`
3. body `metadata.user_id`
4. hash of model plus the first user text

If no stable signal can be derived, `diagnose` may still log structure, but `restore` must skip restoration with reason `no_session_key`.

## Request Diagnostics

Run diagnostics on the exact body that will be sent upstream, after `StripEmptyTextBlocks`.

A request is a candidate when:

- `messages` is present and is an array
- a user message contains one or more `tool_result` blocks
- there is a preceding assistant message with matching `tool_use.id`
- that assistant message lacks a non-empty `type:"thinking"` block
- the final user message is a tool continuation, not a new user question mixed with ordinary text

Diagnostics must not log thinking text or tool input bodies. Log only structural information:

- account ID and account name
- model
- mode
- session key hash
- tool use ID
- assistant block types
- user block types
- action: `logged`, `restored`, or `skipped`
- reason: `missing_thinking`, `cache_miss`, `tool_id_mismatch`, `new_user_turn`, `no_session_key`, or `not_deepseek`

## Request Restoration

In `restore` mode, before building the upstream request:

1. Diagnose the request.
2. Derive the session key.
3. For each missing-thinking candidate, find a cache entry by `sessionKey + toolUseID`.
4. If a matching assistant `tool_use` message exists but lacks thinking, insert the cached `thinking` block before the `tool_use`.
5. If the assistant message is absent but a matching `tool_result` is present, insert the cached assistant message immediately before the tool-result user message.
6. Do not restore if the final user turn includes ordinary text, if tool IDs do not match, or if the cache entry is expired.

The restored order must remain:

```json
[
  {
    "role": "assistant",
    "content": [
      {"type": "thinking", "thinking": "..."},
      {"type": "tool_use", "id": "toolu_x", "name": "...", "input": {}}
    ]
  },
  {
    "role": "user",
    "content": [
      {"type": "tool_result", "tool_use_id": "toolu_x", "content": "..."}
    ]
  }
]
```

## Response Capture

Capture only when DeepSeek thinking continuity is enabled.

For non-streaming responses, parse the Anthropic message response body after it is read and before it is written back to the client. If `content[]` contains both a non-empty `thinking` block and at least one `tool_use` block, cache the assistant content for each tool use ID.

For streaming responses, observe SSE data while still writing the original lines to the client unchanged:

- `content_block_start` opens a block by index and records type.
- `thinking_delta` appends thinking text.
- `input_json_delta` appends tool input JSON for tool blocks.
- `content_block_stop` finalizes a block.
- `message_stop` caches the reconstructed assistant content when it contains thinking and tool_use.

Streaming capture must be best-effort. If parsing fails, log a debug diagnostic and continue passthrough without modifying client output.

## Gateway Integration

Integrate in `forwardAnthropicAPIKeyPassthroughWithInput` after empty text stripping:

```go
input.Body = StripEmptyTextBlocks(input.Body)
input.Body = s.applyDeepSeekThinkingContinuityBeforeForward(ctx, c, account, input.Body)
setOpsUpstreamRequestBody(c, input.Body)
```

Non-streaming capture integrates in `handleNonStreamingResponseAnthropicAPIKeyPassthrough` after reading the upstream body and before `c.Data`.

Streaming capture integrates in `handleStreamingResponseAnthropicAPIKeyPassthrough` alongside the current SSE usage parsing. It must not change the written SSE line.

Do not add a failed-400 retry in the first implementation. Request-before-forward restoration is simpler and avoids duplicate upstream side effects.

## Frontend Configuration

Add a small account advanced option to create/edit account dialogs.

Show the option only for:

- platform `anthropic`
- type `apikey`
- Anthropic passthrough enabled

Use a segmented/select control with:

- Off
- Diagnose
- Restore

Save as `extra.deepseek_thinking_continuity_mode`. Remove the field when `off` is selected.

Add constants in `frontend/src/constants/account.ts` matching backend values.

## Testing

Backend tests:

- account helper defaults to `off`
- non-DeepSeek base URL ignores the mode
- `diagnose` does not modify request body
- `restore` inserts a missing thinking block before matching tool_use
- `restore` inserts a cached assistant message before tool_result when the assistant message is missing
- restoration skips mixed new-user text turns
- restoration skips mismatched tool_use_id
- restoration skips when session key is unavailable
- non-streaming response capture stores thinking and tool_use
- streaming response capture stores thinking and tool_use from SSE events
- `StripEmptyTextBlocks` does not remove thinking blocks

Frontend tests:

- create/edit modal persists `deepseek_thinking_continuity_mode`
- selecting Off removes the extra field
- control is visible only for Anthropic API Key passthrough accounts

## Rollout

1. Ship backend with `off`, `diagnose`, and `restore`.
2. Enable `diagnose` on one DeepSeek Anthropic passthrough account and verify diagnostics.
3. Enable `restore` on that account.
4. Watch for reduced DeepSeek `content[].thinking` errors and absence of unexpected request-body mutation logs.
5. Consider Redis-backed cache only if multi-instance routing makes cache misses common.

## Open Decisions

- The first version will not retry after DeepSeek returns this 400. If request-before-forward restoration is insufficient, add a single guarded retry later.
- The first version uses process-local memory. Cross-instance continuity is explicitly deferred.
