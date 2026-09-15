# Benches — what is measured, and how

Nothing about the prompt, the tools or the model changes without a before/after on these.

## Tool-choice rate bench (`eval:agent-rates`)

- **Cases**: a file of prompts that failed in real use, each with the expected tool (or
  `none`) and, for multi-round cases, scripted tool results and confirmations.
- **Protocol**: N passes per case (default 10) — one pass cannot tell a fix from a
  coincidence; tool chosen or not, latency, tokens, breakdown of failures (`no_call`,
  wrong tool, bad arguments, timeout). `--model a,b` compares models without touching the
  config; results as JSON, `--compare a.json b.json`.
- **With `--record`**: successful passes are written to the learning corpus tagged `bench`;
  five *measure* cases are frozen and never trained on.
- Reference figures: gemma4:12b 92–93 % with persona; the first fine-tune 80 %.

## Multi-round scenarios (`eval:agent-scenarios`)

Long chains, refusals, dead ends without a tool, comparison + memory, reasoning on a result —
on the real loop with fake tool results and scripted confirmations. Reference: 80 %, never
inventing a capability, never re-asking after a refusal.

## Blind voice sample (`eval:voice-sample`)

Same persona prompt (stable portrait via `--memory`), open questions, answers from A/B/C
shuffled with a separate key. The only criterion the benches do not measure is the one the
person decides here. It chose gemma4 over gpt-oss; it preferred the fine-tuned variant 3/3
while the rate bench showed it had lost 13 points on tools. Run both.

## STT bench (`wakeword:stt-bench`)

A frozen corpus of 20 real commands × 3 synthetic voices; three prompts on identical audio;
word error rate and commands understood. It showed the 56-term vocabulary list made the
fast model worse than no prompt (23/60 vs 24/60) and example sentences better (42/60).
Never resynthesise the corpus casually: it invalidates comparison with earlier passes.

## Wake-word evaluation (`wakeword:eval`, `wakeword:promote`)

Frozen validation sets (the user's recordings, never trained on), threshold equal to the
service's, seeded and **deterministic** (it was not, once: 3, 6 then 5 misses for the same
model). Four promotion conditions; a candidate is scored in shadow by the live service before
promotion; rejects kept with their measurement. The bench is what makes nightly training safe.

## Learning capture

- `fine_tune_log`: input, output, model that answered, session; feedback optional
  (`accepted` / `corrected` + text / `rejected` + reason). Thumbs on the bubble or three typed
  commands. **Unjudged rows are excluded from every export.**
- `tool_turn_log`: the whole transcript of a turn with tool calls (system prompt, tools, all
  messages as the model saw them), only on a natural end of turn; the user's confirmations
  as labels (all approved → accepted, one refused → rejected), a thumb overrides.
- Exports: SFT (chat format with `tools`) and preference (`prompt`, `chosen`, `rejected`).
  Deep research adds a preference pair per attempt (local reading vs cloud reading of the same
  sources) and, since 12/09, one pair per observation the local reading missed.
- First cycle, reference machine: prepare → QLoRA (343 s, 11 GB, max sequence 12,288) → GGUF →
  `ollama create` → rate bench: ~15 minutes.
