# Journals — the record is the application's, not the model's

Two append-only JSONL files in the user-data folder. Both are read by the watchdog, by the
benches, and by a human diagnosing an incident. **Every incident in the reference project was
found by reading them, never by guessing.**

## `orchestrator-trace.jsonl` — everything

One line per event, `{"ts": ISO, "type": …, "sessionId": …, …}`. Event types in the reference
instance (counts over ten days in parentheses, for scale): `request` (113), `tool_route`,
`agent_round` (159), `tool_action` (87 — tier + outcome `approved`/`declined`), `tool_call_rejected`
(invalid arguments, with the raw arguments), `delegation_result`, `final` (109),
`memory_judge_skipped`, `memory_decision`, `memory_write`, `memory_write_explicit`,
`memory_flush`, `escalation` / `escalation_result`, `agent_loop_failed`, `agent_loop_cancelled`,
`empty_final_after_tool_detected` / `empty_final_recovered`, `emergency_stop`, `voice_chat`,
`voice_routine`, `deep_research`, `research_judge_failed`, `research_pairs`, `cleanup_executed`,
`outfit_proposal`, `report_action`, `quick_action_request` / `quick_action_final` (retired 15/09).

Rule: **every real execution path writes here** — the conversation panel, the command bar,
the voice, the research pipeline. Two failures of the bar were undiagnosable until it did.

## `kora-intentions.jsonl` — before the question

```json
{"ts":"2026-09-07T14:30:08.249Z","type":"tool_intent","sessionId":"…","orb":"kora","toolName":"web_search","args":{"query":"qu'est-ce que le protocole MPRIS"}}
```

Written the moment the model emits a tool call, **before** the tier is applied and before
any bubble is shown. So a refused call, a thread closed mid-confirmation, or a crash still
leave the intent on disk. It is the file to read when the user asks "what did she try to do?".

## What is *not* in the journals

- Secrets: masked at the main-process boundary before anything is logged.
- The user's mail: subjects and bodies never enter a prompt, so they never enter a trace.
- The model's reasoning tokens: kept per message in the conversation store (foldable in the
  UI), not replayed into the next prompt, not in the trace.

## Conversation store (SQLite in the reference instance)

`conversations` (several threads per orb) · `conversation_messages` (role, content, tool names
used, thinking) · `delegation_results` (provider, task, result — every paid answer) ·
`fine_tune_log` and `tool_turn_log` (see `bench.md`) · `api_usage_log` (every call: provider,
model, tokens, cost, status — the source of every public figure).
