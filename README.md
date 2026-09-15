# Kora — a local agent you can trust

[![Licence: CC BY 4.0](https://img.shields.io/badge/licence-CC%20BY%204.0-lightgrey.svg)](LICENSE)
[![Type](https://img.shields.io/badge/type-reference%20architecture%20%2B%20living%20guide-blue.svg)](#what-is-in-this-repository)
[![Method](https://img.shields.io/badge/method-socle-green.svg)](https://github.com/Kairos-commu/socle)

> **Résumé en français** — Kora est une assistante qui tourne sur *une* machine, pour *une*
> personne, sur un petit modèle local (12 milliards de paramètres), avec un vrai accès à ses
> fichiers, son courrier, sa voix et sa mémoire — et à qui il est interdit, par construction,
> de dire qu'elle a fait quelque chose qu'elle n'a pas fait. Ce dépôt ne contient pas le code
> de l'instance de Florent (privé) : il décrit **une Kora générique** — l'architecture qu'on
> peut reconstruire — et tient le **guide vivant** des décisions qui l'ont rendue digne de
> confiance, chacune avec l'incident qui l'a imposée et le mécanisme qui la tient. L'article
> d'intention, en français : [Ce qu'on tente ici](https://mecanique-invisible.com/ce-quon-tente-ici.html).

A **Kora** is a desktop agent that lives on one ordinary machine, learns from one person, and
is trusted with a real slice of that person's life: files, mail, music, voice, memory. It runs
on a small local model (12B, through Ollama), calls tools with real side effects, delegates to
cloud models when its own is not enough — every call counted and capped — and never speaks
first. What makes it trustworthy is not the model and not the prompt: it is a handful of
mechanisms in the application, each of which exists because the model broke something real.

This repository is that pattern, written down so someone else — or their coding agent — can
build their own, with their voice, in their room: the contracts to build from, the guide of
what it cost us to learn them, and the addenda as it keeps moving. If you were told « find it
on GitHub and build on it », start with [`BUILD.md`](BUILD.md).

## What is in this repository

| File | What it is |
|---|---|
| [`GUIDE.md`](GUIDE.md) | **The living guide** — twelve decisions, in the order you will meet them, each with *what we do*, *the incident that forced it*, *how it is held*, *what to copy*. Dated by section, never rewritten: it receives addenda. |
| [`BUILD.md`](BUILD.md) | **How to build one from here** — for a person or a coding agent: reading order, build order with a *done when* per step, what to keep and what to change. |
| [`spec/`](spec/) | **The contracts**, extracted from the reference instance: the 21 tool definitions (JSON, loadable as-is), the tier policy, the assembled prompt, the loop's budgets and guards, the journals, the memory file and its write schema, the benches, the state export. |
| [`architecture.md`](architecture.md) | **A generic Kora** — the components, what each one holds, and what is replaceable. No code; the shape. |
| [`addenda/`](addenda/) | Dated **proposals** of addenda, produced by a daily pass over the project's own documentation. A proposal becomes part of the guide when a human pastes it and commits. |
| [`state.json`](state.json) | A **snapshot of the real instance** — model, orbs, capabilities by function and status, calls and cost, corpus size — exported by the application, never typed. Live version on the [published guide](https://mecanique-invisible.com/notes/building-a-local-agent-you-can-trust.html). |

## The seven invariants of a Kora

These are the properties that make the thing a Kora rather than another desktop assistant.
Each one is held by a mechanism, not by a sentence in a prompt — the guide says which.

1. **It cannot claim what it did not do.** Every capability is a real tool; the final answer
   exists only after the tool's real result is in the transcript. There is no "I did it" path.
2. **The model never judges its own actions.** What it may do, what a cleanup will touch, what
   counts as a valid call — decided by code, with the model as the requester.
3. **Three risk tiers and two journals.** Read/search runs alone; open/launch/sensitive-read
   asks; write/delete is irreversible and asks in red. Every intent is journaled *before* the
   question is asked, so a refusal leaves a trace.
4. **Nothing happens without the person knowing.** A memory write pulses, chimes, and grows a
   visible crystal. Spending is capped per day. A watchdog the model cannot see counts.
5. **Memory is a text file the person can correct.** A portrait written by the person, then
   sections the model may append to through a schema it cannot escape — and a periodic re-read
   of quiet conversations instead of a per-turn judge.
6. **Everything that learns, learns from one person, locally.** The wake word from their voice,
   the command vocabulary from the phrases they actually said, the fine-tune corpus from their
   turns and their confirmations. None of it leaves the machine (a Raspberry Pi counts as the
   machine).
7. **The cloud is a counted recourse, not a default.** A closed set of providers described by
   use; context assembled by the app; every paid answer kept and searchable; one daily cap.

A Kora also does three things that pull in different directions — it **researches** (and may
conclude that nothing emerges), it **assists** (and must answer), it **plays** (and may
invent) — and keeps them on separate paths rather than one prompt. That is a design decision
too; the article explains why.

## What this is not

- Not an installation guide, and not a product. The reference instance is private because it
  is one person's data through and through. The pattern is public.
- Not "local-first" as a dogma. The local model is the starting point; the cloud is measured.
- Not another orb-with-a-voice. Several open-source projects already do the visual part as
  well or better; what is rare is the [incident journal](https://mecanique-invisible.com/notes/tool-calling-failure-modes.html)
  and the mechanisms that came out of it.

## How the guide lives

- **Sections are dated, not rewritten.** A change of decision or a new measurement becomes an
  addendum under its section, with its date; the *Revisions* log at the bottom of `GUIDE.md`
  says what moved.
- **Addenda are proposed, then pasted.** Once a day, a pass over the project's own dated
  documentation (every change in the project writes its paragraph in the docs — that is what
  makes this possible) produces a proposal in [`addenda/`](addenda/), each sentence tagged
  `[code-verified]` or `[doc-only]`. Nothing enters the guide, and nothing is published,
  without a person pasting it. A capability is never cited as working unless it has been seen
  working in real use.
- **Figures are exported, not typed.** `state.json` and the live banner come from the
  application's own `etat:export`.

## Relation to *socle*

[Socle](https://github.com/Kairos-commu/socle) is the **method**: the guardrails and working
practices for building software with a coding agent over the long run — tagged verification,
mechanical guards, a function registry, an incident journal, thirteen guardrails for an agent
that actually acts. It is generic and stack-free.

This repository is the method **applied to one thing**: an agent that acts on a person's
machine. Section 11 of the guide is where the two meet. If you build a Kora, socle is what
keeps your build honest; if you use socle, a Kora is what it looks like when the agent being
built is itself an agent.

## Author, licence

Florent Klimacek — [mecanique-invisible.com](https://mecanique-invisible.com). Text under
[CC BY 4.0](LICENSE): reuse it, quote it, build from it, say where it came from.
