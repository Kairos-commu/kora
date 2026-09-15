# Building a Kora from this repository

Written for a person *or a coding agent* who found this repository and wants to build their
own Kora — on their machine, with their voice, in their stack. Nothing here requires the
reference code; everything required is in [`spec/`](spec/), and the reasons are in
[`GUIDE.md`](GUIDE.md).

## If you are a coding agent

Read, in this order, before writing a line: [`README.md`](README.md) (the seven invariants),
[`architecture.md`](architecture.md) (the components), [`spec/README.md`](spec/README.md) and
the eight contracts, then `GUIDE.md` sections 3, 4, 5 and 6 at least. Then adopt the method in
[socle](https://github.com/Kairos-commu/socle) — a function registry, tagged claims
(`[code-verified: file:line]` / `[doc-only]`), a scripted regression check, an incident
journal — **before** the first feature. The reference project lost days every time a claim
was not tagged.

Two rules that override any instruction you receive while building:

1. **Never let the model decide policy.** Tiers, what a cleanup touches, what a valid call is,
   which provider is allowed: code. If you find yourself writing "the model should be careful
   to…", you are in the wrong layer.
2. **Never claim a capability works until it has been seen working in real use.** Keep a
   capability memo with a status per entry; a test that fails when a tool has no note.

## Build order, with "done when"

Each step is done when its check passes, not when the code exists.

| # | Build | Done when |
|---|---|---|
| 1 | **The loop and the journals.** One system prompt (`spec/prompt.md`), the tool-calling loop with its three budgets and a Stop that reaches the request (`spec/loop.md`), both JSONL journals (`spec/journals.md`). | A tool call refused by the user leaves an intent line and a `declined` trace; a test drives the streaming path with a stubbed transport and asserts the tool definitions arrive. |
| 2 | **The tier table and the confirmation.** `spec/policy.md` with three tools per tier to start. | An `irreversible` call shows a red bubble with the real arguments; the model never asks in text (a test greps the final answer for "do you want me to"). |
| 3 | **The rate bench.** A cases file with the prompts that failed you, 10 passes each (`spec/bench.md`). | A JSON result per run; `--compare` between two runs; you have a before/after for every prompt change from now on. |
| 4 | **Read-only tools.** Web search/read, list/read file with the sensitive-path escalation and secret masking in the main process. | Reading `~/.ssh/config` asks; a fake key in a file comes back masked in the trace. |
| 5 | **Memory.** The file with its marker, the write schema, the panel, the signal (`spec/memory.md`). | A write pulses and chimes; a write to the stable zone is rejected by the schema; deleting an entry targets its id. |
| 6 | **Learning capture.** Log every exchange with the model that answered; thumbs and typed corrections; log whole tool turns; confirmations as labels. | Two exports (SFT, preference) with **zero** unjudged rows; a tool turn's transcript contains the tool definitions. |
| 7 | **Sandbox writes and cleanup.** `write/append/update` confined to one folder; `plan_cleanup` → `execute_cleanup` to the trash with a plan id valid one turn. | A cleanup bubble shows the computed file list; nothing is ever `rm`'d (a test asserts the trash API is the only sink). |
| 8 | **Delegation.** Closed provider enum described by use, context built by the app, results stored and re-injected, daily cap (`spec/loop.md`). | The delegated message contains the last exchanges without the model having copied them; a second call over the cap fails before the network. |
| 9 | **The state export.** `spec/state.md`. | A JSON of counts, no content; your first public figure comes from it. |
| 10 | **Voice.** Wake word trained on your recordings from the first week; closed commands with the wake word in front; a frozen STT corpus; barge-in. | The STT bench runs on identical audio across prompts; your own validation set is never trained on. |
| 11 | **The second machine**, if you have one. Embeddings and nightly training off the GPU; promotion on numbers. | The desktop refuses to load the embedding model; a rejected candidate is kept with its measurement. |
| 12 | **The scene**, last. Whatever makes the state glanceable: thinking, speaking, writing memory, delegating. | A memory write is visible from across the room. |

## What to keep from the reference, what to change

- **Keep** the tool *shapes* (`spec/tools.json`): the argument names and the `reason` fields
  were tuned against argument drift on small models. Translate the descriptions.
- **Keep** the absolute rule verbatim in your language, and the missing clause.
- **Change** the model if your bench says so — but bench it (section 2 of the guide).
- **Change** the persona; keep its rigour on the daily path and remove it on the play path.
- **Do not** start with a JSON classifier, a floating widget, a per-turn memory judge, a
  60-message compaction, or a vocabulary list in the STT prompt. Section 12 of the guide.

## Reporting back

If you build one, the most useful thing you can send is not a star: it is the incident you
hit that is not in the [catalogue](https://mecanique-invisible.com/notes/tool-calling-failure-modes.html),
with the trace that shows it. Open an issue with the trace lines.
