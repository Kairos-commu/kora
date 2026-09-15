# Addenda — proposed, then pasted

Each file here is one day's **proposal** of addenda to [`GUIDE.md`](../GUIDE.md), produced by
a pass over the reference project's own dated documentation (every change in that project
writes its paragraph in `docs/`; the pass reads the diff since the previous pass, keeps what
changes a decision or adds a measurement, and rejects the rest with a reason).

Rules the pass follows, and that a human reviewer should hold it to:

- a capability is cited as working only if its status in the application's memo is
  *seen working in real use* — never *never re-checked*;
- every sentence of fact carries `[code-verified: file:line]` or `[doc-only]`; the tags are
  removed when pasted;
- a figure comes from a named file;
- nothing about the research experiment's implementation is proposed before its bench has
  run once.

A proposal becomes part of the guide when a person pastes it under the right section, adds
the *Revisions* line, and commits. Proposals are kept as they were written — including what
was refused and why.
