# Memory — a text file the person can correct

One Markdown file in the user-data folder, never in a repository. Two zones separated by a
**literal marker line** (not a heading the parser has to guess):

```markdown
# <Portrait of the user, written by the user>
…as long as they like: who they are, how they think, what they expect from Kora…

<!-- MÉMOIRE DYNAMIQUE — écrite par Kora, jamais à la main -->

## Décisions d'arbitrage récentes
- [2026-09-07T14:30:08.249Z] Never delegate a media command to the cloud. <!-- id:6f1c…-uuid -->

## Patterns détectés
## Fils ouverts
## Anomalies techniques
```

- **Stable zone** (above the marker): written by the person, editable in the app, injected
  **whole** into every persona. The model has no key for it: a write targeting it is rejected
  by the schema before reaching the file.
- **Dynamic zone**: four sections, fixed names (`arbitrage`, `patterns`, `fils_ouverts`,
  `anomalies`). One entry = one line, ISO date in brackets, text, and a UUID in an HTML comment.
  The file stays readable as-is; deletion targets the id, never the date (two entries in the
  same millisecond share a date).
- **Two projections**: the panel reads everything (up to 30 per section); the prompt receives
  the last 6 per section, without timestamps.

## The write contract

The model never touches the file. It emits a decision validated against this schema (the
memory judge, JSON-constrained), or calls `write_memory` / `update_memory` (tools, `auto`):

```json
{
  "type": "object",
  "properties": {
    "should_write": {
      "type": "boolean"
    },
    "section": {
      "type": [
        "string",
        "null"
      ],
      "enum": [
        "arbitrage",
        "patterns",
        "fils_ouverts",
        "anomalies",
        null
      ]
    },
    "entry": {
      "type": [
        "string",
        "null"
      ]
    },
    "contradicts_entry_id": {
      "type": [
        "string",
        "null"
      ]
    }
  },
  "required": [
    "should_write",
    "section",
    "entry",
    "contradicts_entry_id"
  ]
}```

Validation is two-level: shape, then meaning (`should_write: true` with no section or entry
is rejected). `contradicts_entry_id` lets a new entry supersede an old one instead of piling up.

## When writes happen

| Trigger | Mechanism | Guard |
|---|---|---|
| The person literally asks ("remember this") | `write_memory` tool | Explicit trigger required in the raw message + anti-duplicate against the condensed memory |
| A tool really failed this turn | Memory judge, one JSON call, `low` thinking, temperature 0.1 | Section `anomalies` only; never emptied automatically |
| A thread goes quiet for 20 minutes (or at launch, for threads left unread) | **Re-read of the whole unread part**, one local call, at most 3 items per section | Never during a local model call; one re-read at a time; a `flushed_through` marker distinct from compaction |

The per-turn judge on every answer was removed after measurement (37 calls in a morning, 0
useful writes). The 60-message compaction that was supposed to take over never triggered: no
real thread reaches 60 messages. The re-read replaced both.

## The signal

Every real write: the orb pulses violet, a two-note chime plays, the memory panel refreshes,
and (volumetric rendering) a crystal grows inside the orb at the semantic position of the
entry. A deleted entry dissolves. This is the mechanism behind "nothing happens without my
knowing" — build the equivalent for your surface before building the second memory feature.
