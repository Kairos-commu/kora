# State export — figures read, never typed

`npm run etat:export` in the reference project writes one JSON of **counts** (never a content:
no memory entry, no mail subject, no key, no personal path). The site reads it at build time;
this repository keeps a snapshot as `state.json`. If you build a Kora, build this early: every
public sentence with a number in it will otherwise be wrong within a week.

```jsonc
{
  "generatedAt": "ISO", "generatedAtFr": "15 septembre 2026",
  "app": { "commit": "abc1234", "commitDate": "ISO", "commits": 329, "firstCommitDate": "ISO", "daysSinceStart": 15 },
  "kora": { "model": "gemma4:12b", "orbs": 9, "orbsVisible": 9, "moons": 2, "providers": ["anthropic", "..."] },
  "capabilities": {
    "tools": 21, "triggers": 25, "chains": 10, "selfInitiated": 9, "gaps": 9, "planned": 1,
    "byVertex": { "recherche": 9, "quotidien": 52, "jeu": 16 },          // same counts as the in-app memo header
    "byStatus": { "eprouve": 36, "a-verifier": 24, "limite": 5 },
    "byVertexStatus": { "recherche": {…}, "quotidien": {…}, "jeu": {…} },
    "gapsByVertex": { "recherche": 1, "quotidien": 7, "jeu": 2 }
  },
  "usage": {                                                             // null when the database is absent
    "since": "ISO", "until": "ISO", "days": 10,
    "calls": { "total": 1527, "local": 1148, "cloud": 127, "web": 252, "errors": 15 },
    "costUsd": 1.181,
    "last7d": { "local": 763, "cloud": 225, "web": 0, "costUsd": 0.714 },
    "localModels": ["gemma4:12b"]
  },
  "corpus": { "exchanges": 179, "accepted": 34, "corrected": 50, "rejected": 5, "unjudged": 90, "toolTurns": 5, "koraThreads": 19 },
  "memory": { "dynamicEntries": 17 },
  "wakeword": { "recordings": 177 },
  "voicePrototypes": 88,
  "research": { "notes": 38, "readings": 67, "topics": 17, "nothingEmerged": 2 }
}
```

Sources in the reference instance: the orbs config (model, orbs), the capability memo's
`memoCounts()` (the compiler-checked list of capabilities with status and function), the
SQLite database read-only, the memory file (number of `<!-- id:` lines), the wake-word
recordings folder, the learned-phrases file, the research trace, `git log`.
