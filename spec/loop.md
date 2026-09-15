# The agent loop — budgets, guards, delegation

## One call, all tools

`runOrchestration(userMessage, recentHistory, sessionId, hooks)` → one tool-calling loop on the
local model. The model may call several tools per round; results are pushed as `role: tool`
messages; the loop ends on a round without tool calls. The final text is the answer.

| Budget | Reference value | Why it exists |
|---|---|---|
| Rounds per loop call | 8 | A model that loops on a failing call must stop |
| Rounds per user turn, retries included | 12 | Retry guards each restarted the budget: 32 rounds for one message |
| Time per turn | 120 s, ×2 when thinking is `high` | A deeper level gets fewer rounds otherwise |
| Minimum rounds before a retry is allowed | 2 | A retry on round 1 is a second first try |

**Stop** reaches the in-flight HTTP request (an `AbortController` through IPC to the main
process's fetch), never just the UI. Streamed text before the stop is kept, marked
interrupted, never logged for training, and never triggers a retry or an escalation.

## Guards after the loop, in order

1. **Empty final after a successful tool** → one local retry with no tools; if still empty,
   show the last *paid* tool result verbatim (a delegation, a web page) rather than nothing.
2. **Escalation to the cloud** (Claude, no tools, original request + recent history) only when
   the loop produced nothing or timed out. One escalation in the reference trace of 113
   requests. Argument bugs on the local side used to cause 70 % of escalations — fix the loop
   before widening the net.
3. **Anti-repeat**: a call with the same name + arguments as one refused or already succeeded
   this turn is not executed; the model gets the earlier result back.

## Streaming

The chat path streams (`content` / `thinking` / `tool_calls` chunks over IPC). Thinking of the
final round is kept with the message (foldable), never re-injected. **The one regression to
never repeat**: the streaming path must copy `tools` to the request exactly like the blocking
path — use one options builder for both, and a test that asserts the definitions reach the
transport.

## Context window

Sliding window of 20 messages sent to the model; the full thread stays on disk and on screen.
For Kora's threads, a compaction summary would replace older turns beyond 60 messages —
designed, tested, never reached in real use. The two most recent delegation results of the
thread are re-injected (2,000 characters each) under a marker.

## Delegation

`delegate_to_provider(provider ∈ {claude, redaction, chatgpt, deepseek}, task, save_as?, attach_path?)`

- **Closed enum**, id → orb resolved in a table; the model never names a model.
- **The app builds the delegated message**: last exchanges of the thread (≤ 6,000 chars,
  most recent first) + the user's current request + the model's `task`. The model says *what
  to do*, never recopies content (measured: 0–1 recopies in 10 on two models).
- `redaction` is the local model itself in a long-form persona (its output is the deliverable,
  optionally saved by the app via `save_as` — never re-typed through `write_file`).
- `attach_path` (vision) only on `claude`; a known attachment this turn routes the tool list
  to communication tools only, so the attachment is never "forgotten".
- Every successful result is stored (`delegation_results`) and searchable by
  `recall_provider_answers`; the two most recent are re-injected automatically.
- The result returned to the model is a confirmation text, **never the URL or the raw
  payload** for browser tools (anti-bounce: the model would open it again).
