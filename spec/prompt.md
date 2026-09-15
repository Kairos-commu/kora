# The system prompt — structure, not text

The prompt is **assembled per turn** from sections. Its text is in the language the user
speaks to Kora (French in the reference instance) and addresses the model in the **second
person** ("Tu es Kora…") — the third person ("Kora must…") let the model answer as an
observer of itself. The wording below is what the reference instance uses, translated; the
French originals are what was tuned on the 12B model.

## 1. Header, always present

- Identity and voice: *"You are Kora… you answer the user with your real voice — never a
  technical report on your own workings unless asked."*
- The machine: *"You run on the user's personal PC, Linux; `~` is their home; never ask for
  their username or a full path; never write a Windows path."*
- The tool boundary: *"Your ONLY tools are those listed below. You have no built-in
  `browser` or `python` tool even if your training made you familiar with such tools — here
  they do not exist."* (A model trained with built-in tools will call them from memory.)
- Multi-sentence messages: *"The message often starts with a reaction to what you just said
  before the actual instruction. Read the whole message; act on the instruction even if it
  is not the first sentence."* (Measured: a 9B model dropped an action preceded by a
  conversational sentence.)

## 2. The absolute rule, verbatim (translated)

> ABSOLUTE RULE, THE MOST IMPORTANT: never claim to have done something (delegated to
> another model, read/opened a file or a URL, searched the web, written to memory) without
> having REALLY called the corresponding tool and received its result in this same
> conversation. No shortcut, no simulation, no improvising on a result you have not seen —
> if you did not call the tool, you did not do it; say so honestly rather than invent. This
> holds for the FUTURE too: never end an answer with "I'm launching the call" / "I'll take
> care of it now" / "I'll write it" without having ALREADY included the tool call in this
> same turn — an announced but uncalled tool does not run afterwards; this message would be
> your last chance. Either you call the tool NOW (this turn, not the next), or you say
> clearly that you need a clarification before you can act — never a promise instead of one
> or the other.

The rule is in the prompt *and* held by the loop: there is no path to a final answer that
skips a tool result. The prompt part exists for the wording of refusals, not for enforcement.

## 3. Sections per capability

One section per tool group, in the same order as `tools.md`: attachment (only when a file was
attached this turn), delegate (always), web, PC read, PC actions, PC write, cleanup, memory
(always), past threads (always), provider answers (always). Each section says **what the
tools do, when to call them, and what never to do** — e.g. for PC actions: *"each of these
triggers a confirmation by the application; never ask permission in text yourself"*; for
memory: *"call it only if the user literally asks to remember; when this thread ends (twenty
minutes of silence) you will re-read it in full, apart — that is where a thread's memory is
decided, not turn by turn"*; for past threads: *"never invent a memory of a thread you did
not actually consult"*.

## 4. The missing clause

If a section is omitted this turn (an attachment routes to communication tools only), the
prompt adds: *"What you cannot do THIS TURN (a temporary limit of this turn, NOT a general
limit): … If the user clearly needs it, ask them to rephrase in a new message — never claim
it is impossible in general, that would be false."* Without this clause the absolute rule
turns into a lie about a real capability.

## 5. Memory injection

After the sections: *"What you know about the user"* — the **stable portrait** in full, then
the **condensed dynamic memory** (last 6 entries per section, no timestamps). Then the two
most recent provider answers of this thread, under an explicit marker. See `memory.md`.

## 6. Per-path persona

The command bar and the voice use the same prompt on a profile (`policy.md`). The research
path and the play path use **different** prompts: research forbids invention and honours
"nothing emerges"; play removes the rigour portrait entirely. One prompt cannot serve the
three functions.

## Parameters that go with it (reference instance)

`temperature` 0.3 · `num_ctx` 32768 · thinking as a string (`low` for the bar and the memory
judge, `medium` by default, `high` when the user picks it) · a JSON `format` constraint only on
the three structured calls (memory decision, compaction, research reading), each with a
single-try fallback without `format` because some models return empty content under it.
