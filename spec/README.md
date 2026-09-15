# Specifications — enough to build from

These files are the **contracts** of a Kora, extracted from the reference instance rather
than described from memory. Read them in this order; each says what it holds and where the
number or the rule came from.

| File | Contract |
|---|---|
| [`tools.md`](tools.md) · [`tools.json`](tools.json) | The 21 tool definitions as the model receives them (Ollama tool format), with their tier |
| [`policy.md`](policy.md) | The three tiers, escalation by argument, per-path profiles, what the model and the app must and must not do around a confirmation |
| [`prompt.md`](prompt.md) | The assembled system prompt: header, the absolute rule verbatim, one section per tool group, the missing clause, memory injection |
| [`loop.md`](loop.md) | Budgets, Stop, the guards after the loop, streaming, the context window, delegation |
| [`journals.md`](journals.md) | The two JSONL journals, their event types, the intent-before-question rule; the conversation store |
| [`memory.md`](memory.md) | The memory file format, the write schema, the three triggers, the signal |
| [`bench.md`](bench.md) | The benches every change must pass, and the learning capture |
| [`state.md`](state.md) | The state export — figures read from the app, never typed |

What is deliberately **not** here: the renderer (the orb, the scene), the voice pipeline's
code, the research pipeline's prompts (the experiment it serves is not published before its
bench has run), and anything personal. The shape is in [`../architecture.md`](../architecture.md);
the reasons are in [`../GUIDE.md`](../GUIDE.md).
