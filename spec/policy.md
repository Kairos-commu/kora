# Policy — three tiers, and who asks

The tier of a tool is a **table in code**, not a judgment of the model. The reference table is
in `tools.md`; the rules below are what the application does with it.

## Tiers

| Tier | Runs | Examples |
|---|---|---|
| `auto` | immediately | web search, read a file, list a folder, delegate, write memory, recall |
| `confirm` | after the user says yes in a bubble | open a URL or a path, launch an app, control media, open a site search |
| `irreversible` | after the user says yes in a **red** bubble that shows exactly what will happen | write / update / append a file in the sandbox, execute a cleanup plan |

## Escalation by argument

A tool's tier can be *raised* by its arguments, never lowered:

- `read_file` and `list_directory` on a **sensitive path** (`.ssh`, `.env`, `credentials`,
  keys, tokens…) go to `confirm`. Values that look like secrets in the returned content are
  masked **in the main process**, before the content reaches the renderer or the model.
- `execute_cleanup` only accepts a `plan_id` produced by `plan_cleanup` **in the same turn**;
  the bubble shows the plan's real file list; files go to the system trash, never `rm`.

## Per-path overrides (a profile, not a second engine)

The same engine serves the conversation panel, the command bar and the voice. Each entry
path passes a *profile* that may relax tiers for a short list of tools, and only there:

| Path | Override | Why |
|---|---|---|
| Panel | none | The full conversation; every bubble is normal |
| Command bar (keyboard) | six reversible PC actions run without a bubble | "turn it down" typed in a bar is the confirmation |
| Voice | `control_media` only runs without a bubble | Anything else asks, and the answer can be given by voice |
| After external content (a web page, a file) was read this turn | only `control_media` keeps its override | Text written by a third party must not drive an action |

Profiles also carry: a 45 s cap and thinking `low` on the bar, `escalate: false` (never a
cloud call for a media command), no conversation history.

## What the model must not do

- Ask for permission **in text** ("Do you want me to…?"). The application asks; the model
  calls the tool directly. Two questions for one action is the failure mode.
- Re-issue a call with the same signature after it was **refused** in this turn, or after it
  **succeeded** (anti-repeat guard on tool name + arguments).
- Claim the tool is unavailable when it merely was not shown this turn (the prompt says so
  explicitly; see `prompt.md`).

## What the application must do

- Write the **intent** to the intentions journal *before* asking (see `journals.md`).
- Show the real arguments in the bubble (path, URL, plan contents), not a paraphrase.
- Return a refusal to the model as a tool result ("refused by the user"), so the model can
  answer honestly, and never retry on its own.
- Cap cloud spending per day ($1 in the reference instance) and web-search credits (100),
  checked in the main process before any network call.
