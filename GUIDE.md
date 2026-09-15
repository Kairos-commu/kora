# Building a local agent you can trust — what we learned, in order

This is the technical companion to two other texts: a French article on *what* we are
attempting ([Ce qu'on tente ici](https://mecanique-invisible.com/ce-quon-tente-ici.html)) and the
[catalogue of tool-calling failure modes](https://mecanique-invisible.com/notes/tool-calling-failure-modes.html). It is
written for someone who wants to build the same kind of thing — a small model on their own
machine, trusted with a real slice of their life — and would rather start from our
decisions than from our first week.

It is not an installation guide. The repository is private, and the point is not to run
Kora; it is to know, before you start, which decisions cost us days and what held them in
place afterwards. Every section follows the same shape: **what we do**, **the incident that
forced it**, **how it is held** (a test, a mechanism, a measurement — never a rule in a
prompt), and **what to copy**.

## How to read this document

It is dated by section, not rewritten. When a decision changes, the section gets an
addendum with its date, and the [revisions](#revisions) log at the bottom says what moved.
Figures in the state line below are read from the application (a JSON the app exports —
`state.json` here, the same file on the site at build time) — they are not typed, and they
are not a promise.

<!-- state -->
_Live figures — model, orbs, capabilities by function and status, calls and cost — are read from the application at build time on the [published version](https://mecanique-invisible.com/notes/building-a-local-agent-you-can-trust.html); a snapshot is committed here as [`state.json`](state.json)._

## 1. The hardware you actually have

**What we do.** One desktop GPU with 16 GB of VRAM (an RTX 5060 Ti), 29 GB of RAM, a
Raspberry Pi 4 with 4 GB on a shelf, a USB microphone. Nothing else.

**What that means in practice.** The desktop compositor (GNOME/Mutter) takes between 1.5 and
2.8 GB of the 16 depending on what is open. A 12B model at Q4 with a 32k context takes
roughly 8 to 10 GB resident. Whisper for speech-to-text and Piper for speech share the rest.
So exactly **one large model is resident at a time**, and every feature that wants a second
one has to negotiate. Two consequences shaped the project:

- The quick-command path and the conversation path deliberately share the same model, even
  though a smaller dedicated model would answer faster — swapping models on Ollama costs
  about five seconds each way, measured, and a voice command cannot wait for that.
- The embedding model (bge-m3, 1.2 GB) never runs on the desktop GPU. It would evict Kora's
  model. It runs on the Pi instead (section 9).

**What we got wrong first.** Whisper ran on CPU for a week while the GPU sat there, because
the build we had picked did not use it. Voice latency was declared a "wall" of the project;
it was a build flag. Before calling anything a hardware limit, check what is actually running
where (`nvidia-smi` per process).

**What to copy.** Write down, once, what your GPU holds with the desktop running. Decide up
front which single model is resident, and treat everything else as a guest.

## 2. Choose the model with a bench, not with a rumour

**What we do.** The model behind Kora is `gemma4:12b`, through Ollama, since 5 September.
Before it: `qwen3.5:9b` (too unreliable in sustained real use — a long list of incidents),
then `gpt-oss:20b` for two days.

**How it was chosen.** A bench of eleven configurations on the tool-choice cases that had
failed in real use (`eval:agent-rates`: N passes per case, tool chosen or not, latency,
tokens, failure breakdown), plus a multi-round scenario bench on the real tool loop with
scripted confirmations, plus a **blind voice sample** — the same persona prompt, open
questions, answers shuffled A/B/C, the user picks without knowing which model wrote which.
gemma4 came out at 92–93 % correct tool choice with the persona injected, 80 % on multi-round
scenarios without ever inventing a capability, and it received tool definitions intact —
`required` and `enum` included — where gpt-oss's Harmony template dropped them.

**The parameters that matter, as of 15 September.** `num_ctx` 32,768 (a normal turn weighs
about 12k tokens once tool definitions are really transmitted; 8,192 truncated answers
mid-sentence, 16,384 was tight). Temperature 0.3 on the agent loop — a compromise between the
0.1 of a pure classifier and Ollama's default 0.8: this loop must both choose a tool
reliably and speak with a voice. Thinking level as a *string* (`low`/`medium`/`high`)
because on some models the boolean `think: false` is silently ignored; the user chooses
`high` per message from the composer, never an automatic complexity detector.

**What to copy.** Keep the cases that failed you in a file. Run them before and after every
prompt, tool or model change — ten passes each, because one run cannot tell a fix from a
coincidence. And do the blind voice test: the bench does not measure whether you want to
talk to the thing.

## 3. A tool loop, not a JSON classifier

**What we do.** One call, one system prompt, all of Kora's capabilities as real tools
(web search, file reads, file writes in a sandbox, opening things, delegating to a cloud
model, writing to memory). The model answers in free text, with or without tool calls, and
can only produce its final answer *after* seeing the real result of what it asked for.

**The incident that forced it.** The first design (31 August) was a JSON classification
contract: complexity, mode, confidence, subtasks, escalation flag, direct answer. The model
had to guess the contract right for anything to happen — and when it guessed "I did it"
instead of doing it, nothing in the structure could tell. The absolute rule of the project
("never claim to have done something without having called the tool") is unenforceable on
a classifier. On a tool loop it is structural: there is no path to a final answer that skips
the tool result. The whole classifier was replaced on 1 September and never came back.

**The incident that nearly hid it.** From 3 to 5 September the loop ran **without a single
tool definition reaching the model** — the streaming request path copied `format`, `think`
and `temperature` by hand and forgot `tools`. The model knew its tools only from prose and
improvised the call format. Three days of "the model is bad at arguments" were one missing
line. The guard is one options builder shared by both request paths, and a test that drives
the streaming path with a stubbed transport and asserts the tools arrive. The lesson is
broader than the bug: **weigh the prompt the server actually received** (Ollama logs it)
against what you think you sent, before diagnosing the model.

**Budgets that keep the loop honest.** 8 rounds per loop call, 12 rounds per user turn
including retries (without the second cap, each retry got a fresh budget — up to 32 rounds
for one message), 120 s per turn, doubled when the thinking level is `high`. A Stop button
that reaches the in-flight HTTP request, not just the UI: an abandoned generation must not
keep the GPU busy. Text streamed before the stop is kept and marked interrupted, never
logged for training.

**What to copy.** If your model can call tools, do not make it fill a form first. Make every
capability a tool, and make "I did it" impossible to say without a tool result in the
transcript.

## 4. The model never judges its own actions

**What we do.** Three decisions that could have been left to the model are made by code:

- **Which tools it sees this turn.** A keyword router (pure function, regex per category:
  web, PC read, PC write, PC action) narrows the tool list — precision of tool choice
  degrades with the number of tools visible. It fails open: no keyword matched, all tools.
  The first version was a second, JSON-constrained model call; it was replaced because a
  12–20B model has more variance on that decision than the decision deserves, especially once
  `pc_action` exposes tools with real effects.
- **What a cleanup will delete.** `plan_cleanup(directory, filter)` is computed by code from a
  whitelist of directories and returns a plan id; `execute_cleanup(plan_id)` moves *that
  plan's* files to the system trash — never `rm` — after a red confirmation bubble that shows
  the real list. The model asks; it never enumerates.
- **What counts as a valid call.** Missing or invalid arguments are rejected by the app before
  anything runs (four times in the current trace), with the reason sent back as the tool
  result so the model can retry — instead of a silently wrong action.

**The mirror of this.** The system prompt is *assembled* per turn: a section only appears if
its category was routed in. Found while testing the router: reducing the `tools[]` array was
not enough, because the prose still named every tool and the model called `open_url` from
memory when it was not in the list. Reproduced 3/3. And an excluded category adds an explicit
clause — "you cannot do this *this turn*; never claim it is impossible in general" —
otherwise the anti-fabrication rule turns against itself.

**What to copy.** Anything that looks like a policy decision — what is allowed, what will be
touched, what is a valid request — belongs in code you can test, with the model as the
requester. Deterministic routing beats a smarter router that is sometimes wrong.

## 5. Three risk tiers, two journals, one confirmation

**What we do.** Every tool has a tier in one table: `auto` (read, search, recall memory),
`confirm` (open, launch, control media, and any *read* of a sensitive path such as `.ssh`
or `.env`), `irreversible` (write a file, delete). Confirmation is a bubble in the
conversation; the quick-command bar and the voice path have their own, narrower forced-auto
lists.

**Two journals, written at different moments.** `orchestrator-trace.jsonl` records every
request, route, round, tool call, confirmation outcome, final answer, failure. And
`kora-intentions.jsonl` is written **before** the confirmation is asked — so an intent is on
disk even if the user refuses, closes the thread, or the app dies. In the current trace: 144
intentions, 77 confirmation requests, 33 refused. The first line, on 7 September, is a
request to read `~/.ssh/config`. It was approved; the sensitive-path escalation is why it
had to be asked.

**Secrets at the privilege boundary.** A file read that returns something shaped like a key
is masked on the main-process side, before the content reaches the renderer or the model —
not by asking the model to be careful.

**A daily cap on spending.** Cloud calls are capped at $1 per day and web-search credits at
100, checked in the main process before any network call. Kora does not know the cap exists;
it just gets a tool error.

**A watchdog the model cannot see.** An integrated monitor counts calls, refusals, spend,
repeated signatures, and flags rules mechanically. It is deliberately absent from the prompt.

**What to copy.** The tier table, the intent journal written before the question, and the
idea that the model's own transcript is not the record — the app's journals are.

## 6. Memory: a text file, a contract, a visible signal

**What we do.** Kora's memory of its user is one Markdown file with two zones separated by a
literal marker. Above it, the **stable base**: a portrait written by the user (who they are,
how they think, what they expect), injected whole into every persona — never truncated,
never written by the model. Below it, four **dynamic sections** the model can write to
(decisions, patterns, open threads, technical anomalies), each entry with a UUID in an HTML
comment so it can be deleted precisely (two entries written in the same millisecond share a
timestamp; they do not share an id). The model never touches the file: it emits a validated
JSON decision, the main process appends under the right section. A write to a section that
is not in the enum is rejected before reaching the file — the stable base has no key.

**Two projections.** The panel shows everything (up to 30 entries per section); the prompt
receives only the last 6 per section, without timestamps. Without this split the file would
saturate the context within weeks.

**The signal.** Every real write pulses the orb violet, plays a two-note chime, and — since
the volumetric rendering — grows a crystal inside the orb at the position of the entry's
meaning. This is a user constraint from day one ("nothing happens without my knowing"), not
decoration. In the first days the pulse alone was easily missed while reading the chat; the
chime was added the same week.

**The failure that took a week to see.** Spontaneous writes stopped almost entirely after the
per-turn memory judge was restricted to explicit triggers (it had made 37 calls in a morning
for zero useful writes). The extraction it was supposed to hand over to — a compaction every
60 messages — **never ran**: no real thread reaches 60 messages. The traces showed zero
spontaneous writes for a week. The fix (14 September): when a thread goes quiet for twenty
minutes, one local call re-reads the whole unread part and extracts at most three items per
section. A separate "flushed through" marker, distinct from compaction, so a thread re-read
today keeps its last turns in clear tomorrow.

**What we measured and discarded.** A novelty trigger by embedding distance (is this message
far from what the stable base already says?) did not separate a sentence worth remembering
from a short command, on 64 real user messages. It was not built.

**What to copy.** One human-readable file, a schema the model cannot escape, a visible signal
per write, and a periodic re-read instead of a per-turn judge.

## 7. Delegating to the cloud without paying twice

**What we do.** `delegate_to_provider(provider, task)` sends work to Claude, ChatGPT or
DeepSeek — a closed enum of three, each described by *use* (Claude is the safety net and the
default; ChatGPT when named or for a second opinion; DeepSeek when named or for code), not by
brand. Providers are added when a real use case demands one, never in advance.

**The app attaches the context, not the model.** Measured on the rate bench: asked to forward
a 1,000–3,000-character text it had just read, gpt-oss recopied it 1 time in 10 and qwen3 0
in 10 — while both chose the right tool and provider 100 %. Four successive prompt guards
had failed to change that. So the delegated message is built by code: the last exchanges of
the thread (capped at 6,000 characters), the user's current request, then the model's task
line. Cost: ~1,500 input tokens per delegation — fractions of a cent, against seven ChatGPT
calls for one request in the incident that started this.

**Answers are kept.** Tool results live in the loop's local `messages` array and die with the
call. A provider's 1,775-character answer survived only as the 171 characters Kora quoted —
and the next turn Kora asked the user to remind her what DeepSeek had said. Since then every
successful delegation is stored (provider, task, result), the two most recent are re-injected
into the next turns under an explicit marker (capped at 2,000 characters each — this weighs on
the context every turn), and a `recall_provider_answers(query)` tool searches the rest.

**One escalation left.** Claude is called *instead* of the local model only when the loop
produced nothing at all or timed out — one escalation in the current 113-request trace.
On 4 September, about 70 % of the "safety net" cloud calls were traced to argument bugs
on the local side that had already been fixed elsewhere: paid answers thrown away because
the loop had misformatted a call. Fix the loop before widening the net.

**What to copy.** Closed provider enum described by use; context assembled by the app; every
paid answer persisted and searchable.

## 8. Voice, entirely local

**What we do.** Wake word by openWakeWord in a Python venv, trained locally on the user's
own recordings; speech-to-text by whisper.cpp on the GPU; speech by Piper (ElevenLabs
optional); the audio envelope of the spoken file drives the orb's animation, extracted from
the WAV before playback.

**Closed commands, two axes.** A *name* opens a view ("Kora, mails"), a *verb + number* acts
on a line of the view ("Kora, read 3"), a word navigates. "Kora" always first. No time
windows, no free-form parsing on the command path — the model is not in the loop for a
command.

**What the STT bench showed.** A "vocabulary hint" of 56 useful terms in the Whisper prompt —
the obvious thing to do — made recognition *worse than no prompt at all* on the fast model
(23/60 commands understood with the list, 24/60 with nothing) and turned "Kora" into "Quoi ?"
on the user's real voice. A few example sentences instead: 42/60. Measured on a frozen corpus
of 20 real commands × 3 synthetic voices, three prompts on identical audio. Nothing about the
prompt is obvious; bench it.

**What the wake-word bench showed.** Remove the user's 29 recordings from training: misses go
from 3/29 to 18/29. Add 24 synthetic voices (1,217 clips): back to 14/29 — a real gain, far
from enough. Synthetic voices do not pronounce this user's "Kora". There is no shortcut around
recording your own voice in your own room, and doing it early. A speaker verifier behind the
wake word (a logistic regression on openWakeWord's embeddings, threshold chosen by per-day
cross-validation) rejects ~87 % of other voices; a television in the room, about half.

**Barge-in.** The microphone listens *through* Kora's own speech (residual energy over a
frozen threshold, sustained burst = candidate); any voice interrupts when she is waiting for
an answer, only the wake word interrupts when she is reading. Calibrated against a fake
microphone first; the room came second.

**What to copy.** Closed commands with the wake word in front; a frozen STT corpus you re-run
on every prompt change; your own voice in the wake-word training set from day one.

## 9. The second computer

**What we do.** A Raspberry Pi 4 (4 GB) runs Ollama with `bge-m3` resident (F16). It serves
embeddings for the desktop: ~1 s for a short sentence, ~16 s for a 1,500-character excerpt.
The desktop's Ollama is *forbidden* from loading bge-m3 by construction (the IPC refuses a
local host) — it would evict Kora's model.

**What runs there.** Semantic positions for the orb's rendering (a thought sentence becomes a
point in the sphere; a memory entry becomes a crystal), the semantic distance between
research summaries in the research trace, and — at night — the wake-word retraining: the
previous day's confirmed wakes and sorted false wakes go in, a candidate model comes out
(96 minutes per cycle on the Pi, bge-m3 unloaded for the duration and reloaded after), and
is compared to the serving model on the frozen validation sets. Promotion is never automatic:
the candidate is scored *in shadow* by the live service, and a script prints the four
promotion conditions. The first candidate (night of 14–15 September) was rejected: 11 false
wakes against 10. It is kept with its measurement.

**What to copy.** Put the always-on, low-throughput model on a machine that is not your
GPU. Make the desktop refuse to load it. Train at night, promote on numbers, keep the
rejects.

## 10. Capture for learning from day one

**What we do.** Every exchange with Kora is logged (input, output, which model actually
answered), with optional feedback: 👍/👎 on the bubble, or `corrige : …` / `rejette` /
`accepte` typed in the composer. **Silence is never approval** — an unjudged exchange is
excluded from every export. Since 15 September, each *tool turn* is also kept whole (system
prompt, tool definitions, every message as Ollama saw them), and the user's confirmations
double as labels: all approved → accepted, one refused → rejected. A thumb on the bubble
overrides.

**Two exports.** SFT (accepted + corrected → chat format with `tools`, so the framework's
template writes the tool DSL) and preference (corrected → chosen/rejected pairs). Deep
research feeds both for free: the cross-reading runs on the local model *and* on a cloud
judge on identical input, the cloud version is delivered, the local one is logged as
`output` with the cloud one as `corrected_output` — a preference pair without building
anything. Since 12 September, one pair per *observation* the cloud saw and the local
missed, which is the exact defect the bench measures, cut to a size a model can learn.

**The first training run (15 September).** Base → prepare → QLoRA (343 s, 11 GB) → GGUF →
`ollama create` → rate bench: **fifteen minutes end to end**. Blind voice test: the user
preferred the fine-tuned variant 3/3. Tool-choice bench: 80 % against 93 % for the base.
All losses in the same place — where it should delegate or write a note, it writes the
text itself — because the corpus was 90 % research cross-readings: sixty examples that
teach "write it yourself". The same data improved the voice and degraded the actions. The
variant is not in service. Rule learned: never train the "writes" pile without the "acts"
pile.

**What to copy.** Log from the first day, with the model that answered. Make labels come
from gestures the user already makes (confirmations, thumbs). Never export the unjudged.
Expect your first fine-tune to teach the wrong lesson, and have the bench ready to say so.

## 11. The method that makes the rest possible

Most of the above was found by reading traces, not by reasoning. That only works if the
project is built so the traces are complete and the claims are checkable:

- **Tagged claims.** Every statement of status in a session or a doc is tagged
  `[code-verified: file:line]` or `[doc-only]`. The second is not shameful; it must not
  disguise itself as the first.
- **A function registry**, one file per domain, updated before a task is considered done —
  so the next session reads an index instead of grepping blind.
- **A capability memo in the app**, derived from the tool registry, never copied: a test
  fails when a tool has no note; every entry carries its status (`seen working`, `never
  re-checked`, `known limit`) and the function it serves. The memo's header counts are the
  state banner at the top of this page.
- **A scripted regression checklist** after every change (typecheck → tests → lint → known
  regressions grep → e2e if UI touched → agent harness if prompt/tools touched), and the
  rule that the test count must not drift silently.
- **Parallel sessions assumed, never announced.** Several agent sessions work in the same
  tree; the only commit path is a script that builds a temporary index from HEAD plus *your*
  files, replays the pre-commit hook on that exact tree, commits in compare-and-swap, then
  realigns the shared index. A hook refuses `git add`/`commit`/`stash`/`reset --hard` typed
  directly. Two symmetric incidents (one commit taking 21 files from another session; one
  erasing six) made the rule.
- **An incident journal** (7,500 lines as of mid-September) that is the product, not the
  embarrassment: symptom, what the trace showed, false leads, mechanism, fix, test.

The generic part of this method — doctrine, the four skills, the registry checker, the
pre-commit hook — is published separately as [socle](https://github.com/Kairos-commu/socle).

## 12. What we would do differently

- **Record tool turns from the first day.** Until 15 September the exact transcript the model
  saw (system prompt, tools, calls, results) died with each call. The corpus that matters most
  for an agent was the one we never kept.
- **Weigh the real prompt before blaming the model.** Two days of misdiagnosis were one
  missing option; the server log had the number the whole time.
- **Do not build thresholds nobody reaches.** The 60-message compaction was designed, tested,
  and never triggered by a real thread. The memory it was supposed to extract went unwritten
  for a week.
- **Record your own voice early, and a lot.** Forty recordings in varied conditions would
  have unblocked the wake word in the first week; a thousand synthetic clips did not.
- **Bench the STT prompt.** The "helpful vocabulary list" cost five days of a worse wake word.
- **Skip the floating widget.** A second always-on-top window with its own WebGL context
  doubled every security surface and ate VRAM on the machine's tightest resource. Retired
  after three days; the wake word replaced it better.
- **Start with the tool loop.** The JSON classifier lasted one day in real use and produced the
  founding lie. Nothing from it survived.

## Revisions

- **15 September 2026** — first version, written the day the article of intent was
  published. State banner wired to the app's exported state.
