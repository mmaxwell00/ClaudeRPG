# Architecture — ClaudeRPG

Status: **design document, pre-code.** Nothing in this repo is implemented yet.
This file is the contract that the implementation must satisfy; where it says
"must", treat it as a hard invariant rather than a preference.

Section numbers are stable. `README.md` links to §6, §13, and §14 by number, so
insert new material as sub-sections rather than renumbering.

- [1. Goals and non-goals](#1-goals-and-non-goals)
- [2. Design invariants](#2-design-invariants)
- [3. System architecture](#3-system-architecture)
- [4. Rules engine](#4-rules-engine)
- [5. LLM orchestration](#5-llm-orchestration)
- [6. Models and Docker Model Runner](#6-models-and-docker-model-runner)
- [7. Content pipeline](#7-content-pipeline)
- [8. Data model and persistence](#8-data-model-and-persistence)
- [9. Retrieval](#9-retrieval)
- [10. API surface](#10-api-surface)
- [11. Frontend](#11-frontend)
- [12. Secrets, security, and privacy](#12-secrets-security-and-privacy)
- [13. Staged roadmap](#13-staged-roadmap)
- [14. Content and IP](#14-content-and-ip)

---

## 1. Goals and non-goals

### Goals

Build a single-player, LLM-narrated game in Palladium's Rifts setting that runs
entirely on local hardware. The first milestone is **character creation**, not
play: a complete, rules-legal Rifts character produced through conversation
instead of through forty minutes of table lookups.

Character creation is deliberately first for three reasons. It is the most
mechanically dense part of Rifts, so it forces the rules engine to exist early
rather than being retrofitted. It is self-contained and has a crisp definition of
done — the sheet either validates or it doesn't. And it is the part of the system
where an LLM left unsupervised fails most visibly, which makes it the best
possible test of the boundary described in §2.

### Non-goals

Multiplayer, a game master's screen for human GMs, mobile clients, cloud hosting,
and combat beyond what character creation needs to validate equipment. Rifts
campaign play across dimensions is the eventual ambition, not the first target.

Explicitly out of scope forever: distributing anything that would let someone
play without owning the books. See §14.

---

## 2. Design invariants

These are the rules that, if broken, mean the project has failed at its premise
rather than merely having a bug.

### 2.1 The LLM never owns a number

**The model may narrate, interpret, disambiguate, and suggest. It must never be
the source of truth for any value that the rules define.**

Every attribute roll, skill percentage, M.D.C. total, P.P.E. pool, experience
threshold, attacks-per-melee count, and legality check is computed by
deterministic Python in the rules engine (§4). The LLM's access to these numbers
is read-only: it receives them as structured context and renders them as prose.

This is not stylistic. An LLM asked to "roll 3D6 and apply the Rifts bonus rule"
will produce plausible numbers with a wrong distribution, and it will do so
silently and inconsistently across a session. A character sheet built that way is
not a Rifts character; it is a hallucination shaped like one. The failure is also
cumulative — a wrong attribute at step one propagates into skill bonuses,
carrying capacity, and combat values, and by the time it surfaces the session is
unrecoverable.

Concretely, this means the model is never asked to do arithmetic and is never
handed a tool that returns free text where a number is expected. Dice come from
one seeded RNG in the engine. If a rule needs interpreting, the model chooses
between engine-supplied *options*; it does not invent the option set.

### 2.2 Every mechanical outcome is reproducible

One seeded RNG, owned by the engine, with the seed persisted per character. Given
a seed and an ordered list of player decisions, the same sheet must come out.
Without this, no bug in character creation is ever debuggable, because you cannot
distinguish a rules error from an unlucky roll.

Corollary: every roll is journaled — what was rolled, which rule invoked it, what
modifiers applied, and the result. The journal is a first-class artifact, not a
log line (§8.4).

### 2.3 The rules engine is testable without a model running

The engine is a pure library. It imports no LLM client, opens no sockets, and its
test suite runs in CI with no model present. If testing a rules change requires
booting a 7-billion-parameter model, the boundary has been violated.

### 2.4 No licensed text is ever committed, and the guard is mechanical

Policy alone is insufficient. §14 defines the enforcement: `.gitignore`, a
pre-commit hook, and a CI check. The invariant is that *forgetting* must be safe.

### 2.5 Rules content is data, not code

O.C.C. definitions, skill lists, and equipment live in structured data files
loaded at runtime, never in Python literals. Two reasons: the data derives from
copyrighted material and must stay isolated behind the boundary in §14, and
homebrew content should require no code change.

---

## 3. System architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  HOST (macOS, Apple Silicon)                                         │
│                                                                      │
│   ┌────────────────────────────────────────────────────────┐         │
│   │  Docker Model Runner  — HOST-NATIVE, NOT A CONTAINER   │         │
│   │  chat model + embedding model, Metal GPU  (see §6)     │         │
│   └────────────────────────▲───────────────────────────────┘         │
│                            │ OpenAI-compatible HTTP                  │
│                            │ model-runner.docker.internal            │
│  ┌─────────────────────────┼────────────────────────────────────┐    │
│  │  DOCKER COMPOSE         │                                   │    │
│  │                         │                                   │    │
│  │  ┌───────────┐   ┌──────┴───────────────────────┐           │    │
│  │  │ frontend  │──▶│ backend (FastAPI)            │           │    │
│  │  │ Next.js   │   │                              │           │    │
│  │  │ :3000     │   │  ┌────────────────────────┐  │           │    │
│  │  └───────────┘   │  │ orchestrator      §5   │  │           │    │
│  │                  │  │  - turn loop           │  │           │    │
│  │                  │  │  - tool dispatch       │  │           │    │
│  │                  │  │  - prompt assembly     │  │           │    │
│  │                  │  └───────┬────────────────┘  │           │    │
│  │                  │          │ in-process calls  │           │    │
│  │                  │  ┌───────▼────────────────┐  │           │    │
│  │                  │  │ rules engine      §4   │  │           │    │
│  │                  │  │  PURE — no LLM, no net │  │           │    │
│  │                  │  │  dice / validate / calc│  │           │    │
│  │                  │  └───────┬────────────────┘  │           │    │
│  │                  │          │                   │           │    │
│  │                  │  ┌───────▼────────────────┐  │           │    │
│  │                  │  │ retrieval         §9   │  │           │    │
│  │                  │  └───────┬────────────────┘  │           │    │
│  │                  │  :8000   │                   │           │    │
│  │                  └──────────┼───────────────────┘           │    │
│  │                             │                               │    │
│  │                  ┌──────────▼───────────────┐               │    │
│  │                  │ postgres + pgvector      │               │    │
│  │                  │ characters, journal,     │               │    │
│  │                  │ chunks, embeddings   §8  │               │    │
│  │                  └──────────▲───────────────┘               │    │
│  └─────────────────────────────┼───────────────────────────────┘    │
│                                │ writes, offline, run by hand       │
│              ┌─────────────────┴──────────────────┐                 │
│              │ content-pipeline  §7               │                 │
│              │ PDF → chunks → rules data + vectors│                 │
│              │ reads from ./private/ (gitignored) │                 │
│              └────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.1 Why the rules engine is in-process, not a service

The orchestrator calls the engine as a Python library, not over HTTP. A network
hop between them would buy nothing — they scale together, deploy together, and
have no independent consumers — while adding serialization, a failure mode, and
latency to the hottest path in the turn loop. The boundary that matters is the
*dependency* boundary in §2.3, which a module boundary enforces just as well.

### 3.2 The content pipeline is offline and never runs in the request path

It is invoked by hand, produces artifacts, and exits. It is the only component
that touches licensed PDFs, which keeps the IP blast radius to one directory and
one process (§14). Nothing in the serving path can reach the source material.

---

## 4. Rules engine

Pure Python. The system of record for all mechanics.

### 4.1 Responsibilities

- **Dice.** One seeded `random.Random`. Palladium notation (`3D6`, `1D4+1`,
  `2D4x10`) parsed and evaluated in one place, including the exploding-roll
  behaviour Rifts uses for high attribute rolls.
- **Attributes.** The eight Palladium attributes (I.Q., M.E., M.A., P.S., P.P.,
  P.E., P.B., Spd), their rolling rules, and the derived bonuses each one grants.
- **Character classes.** O.C.C. and R.C.C. definitions loaded as data (§2.5):
  attribute prerequisites, granted skills, related-skill allowances with their
  category restrictions, secondary skills, starting equipment and money.
- **Skills.** Base percentages, per-level advancement, attribute-derived
  bonuses, and the distinction between O.C.C. skills, related, and secondary.
- **Derived combat values.** Hit Points, S.D.C., M.D.C. where the race or armour
  provides it, attacks per melee round, and the strike/parry/dodge bonus set.
- **Resource pools.** P.P.E. and I.S.P. where the class grants them.
- **Validation.** The important one. Given a sheet in any state, return the list
  of rule violations and the list of legal next choices.

### 4.2 The validation model

Validation is the interface the orchestrator leans on hardest, so it returns
structure rather than prose:

```python
@dataclass(frozen=True)
class Violation:
    rule_id: str        # stable id, e.g. "occ.attr_prereq.iq"
    severity: Severity  # BLOCKING | WARNING
    message: str        # human text, for the model to paraphrase
    field: str | None   # sheet path, e.g. "skills.related[2]"
    citation: str | None  # book + page, for the player's own lookup

@dataclass(frozen=True)
class Choice:
    field: str
    options: list[Option]   # the ONLY legal values
    min_picks: int
    max_picks: int
```

`Choice` is what keeps §2.1 honest. The model never proposes a skill; it asks the
engine what is legal, then narrates the options the engine returned. A sheet is
complete when validation returns no `BLOCKING` violations and no unfilled
required `Choice`.

Rule ids are stable strings because they are the join key between a violation,
its test case, and its citation. When a rule turns out to be wrong, the id is how
you find every place it was applied.

### 4.3 Character creation as a state machine

Creation is an explicit ordered state machine, not a conversation that happens to
converge. Each state declares what it needs, and the engine can always answer
"what now?" — which is what makes the process resumable and testable.

```
attributes → race/R.C.C. → O.C.C. → skills → equipment
          → cybernetics (if applicable) → alignment → background → COMPLETE
```

Transitions are gated by validation. The player may revisit an earlier state; the
engine recomputes everything downstream and reports what the change invalidated,
which it can do precisely because of the journal in §8.4.

### 4.4 Testing

Table-driven tests over known-good characters, each pinned to a seed and a
decision list. The suite must include characters built by hand from the books as
fixtures, because the only real oracle for "is this a legal Rifts character" is
the source material. Fixtures record *derived* values and rule ids — never
transcribed rule text (§14).

---

## 5. LLM orchestration

### 5.1 The turn loop

```
player utterance
  → assemble context: sheet state, pending Choice set, retrieved passages (§9)
  → model call with tools
  → tool calls execute against the rules engine (deterministic)
  → engine returns new state + violations + next Choice set
  → model call to narrate the outcome
  → persist journal entry, emit to client
```

Two model calls per turn, not one: the first decides what the player meant and
which tools to invoke, the second narrates results it cannot alter. Splitting
them is what makes §2.1 structurally true rather than merely requested in a
prompt — by the time the model writes prose, the numbers are already fixed and
persisted.

### 5.2 Tools

The tool surface is deliberately narrow, and no tool accepts or returns a
free-text number:

| Tool | Purpose |
|---|---|
| `get_sheet` | Current character state |
| `get_pending_choices` | Legal options at this step |
| `apply_choice` | Record a player decision; returns new state + violations |
| `roll` | Invoke a rules-defined roll; engine picks the dice, not the model |
| `validate` | Full violation list |
| `lookup_rule` | Retrieval over rules corpus (§9) |
| `revise` | Return to an earlier creation state |

`roll` takes a *rule id*, never a dice expression. The model cannot ask for
`3D6`; it asks to perform `attributes.roll.iq` and the engine decides what that
means. This closes the obvious hole in §2.1.

### 5.3 Persona and prompt structure

System prompt in three layers: setting voice, the mechanical contract ("you do
not compute values; you invoke tools and narrate results"), and the current
creation state. Only the third layer changes between turns, which keeps the
prefix stable for prompt caching.

### 5.4 Failure handling

A local 7–14B model will produce malformed tool calls. The orchestrator must
treat this as normal: validate every tool call against its schema, retry with the
error appended, and after N failures fall back to rendering the pending `Choice`
set as a plain UI form. **Character creation must always be completable without
the model cooperating.** The LLM is the preferred interface, not the only one —
and building the form fallback early also gives the rules engine a consumer that
never lies about what it received.

---

## 6. Models and Docker Model Runner

### 6.1 The host-native execution model

On macOS, Docker Model Runner does **not** run inference in a container. It runs
as a host-native process so it can reach the Metal GPU directly. Models execute
on the host; only the rest of the stack is containerized. This is expected
behaviour, not a misconfiguration.

### 6.2 The consequence for compose

Because the model is on the host and the backend is in a container, the backend
must reach *out*. Inside compose, the endpoint is:

```
http://model-runner.docker.internal/engines/v1
```

For a backend running natively on the host instead (the fast dev loop), it is
`http://localhost:12434/engines/v1`.

**This single difference is the most likely thing to burn an evening.** Make the
base URL a required environment variable with no default, so a missing value
fails loudly at startup instead of silently falling back to localhost inside a
container and producing a connection error three layers deep in a turn. A startup
health check that calls `/models` and refuses to serve on failure is worth
writing on day one.

### 6.3 Model roles

| Role | Requirement | Notes |
|---|---|---|
| Chat / narration | Reliable tool-calling, long context | Tool-call reliability matters more than prose quality; §5.4 exists because this will still be imperfect |
| Embedding | Consistency over quality | Changing this model invalidates the whole vector store (§9.3) |

Both are served by Model Runner over an OpenAI-compatible API, so the client is
one thin wrapper and the model choice stays configuration.

### 6.4 On cloud escalation

Deferred, not designed out. If narration quality proves insufficient locally, the
escalation boundary is clean: only the narration call in §5.1 could go to a
frontier model, never the rules engine and never retrieval. Note that escalating
narration means sending retrieved passages of licensed material to a third party,
which is a §14 decision before it is a technical one — and the reason this is
deferred rather than built.

---

## 7. Content pipeline

Offline. Converts purchased PDFs into two distinct outputs with very different IP
status. Understanding that distinction is the whole design.

### 7.1 The two outputs

**Rules data** — structured facts. `{"occ": "...", "iq_min": 12}`. Facts about a
game system, expressed in our own schema. This is what the engine loads (§2.5).

**Retrieval corpus** — chunked prose from the books, kept for §9 so the model can
answer "what does the book actually say about this?"

The second is a verbatim derivative of copyrighted text. **It never leaves the
machine and is never committed.** The pipeline writes it to a gitignored path and
the guard in §14 backs that up mechanically.

### 7.2 Stages

```
./private/pdfs/*.pdf   (gitignored, user-supplied, never in git)
  → extract      text + layout; Rifts is dense multi-column with tables
  → segment      chapter / O.C.C. / table boundaries
  → classify     narrative prose vs. mechanical table
  → structure    tables → rules data     (schema-validated)
  → chunk+embed  prose → retrieval corpus (vectors)
  → load         rules data → ./data/ ; corpus → postgres
```

Extraction is the hard stage and will need manual correction. Budget for a
human-in-the-loop review step rather than expecting a clean automated pass:
Rifts stat blocks defeat naive table extraction, and a silently mangled table
becomes a wrong rule that surfaces much later.

### 7.3 Idempotence and provenance

Keyed on PDF content hash so re-running is cheap and safe. Every rules-data
record carries a provenance pointer (source hash, page) so a suspect value can be
traced back to the page it came from — the practical fix for §7.2's error mode,
and the reason `Violation.citation` in §4.2 can exist at all.

---

## 8. Data model and persistence

Postgres with pgvector. One service covers relational state and vector search;
running a second store for a single-player local game would be unjustified
operational surface. SQLite plus `sqlite-vec` is a legitimate alternative if
dropping the container is worth more than the migration path.

### 8.1 Rules data (files, not database)

Version-controllable, diffable, reviewable — but **only** the homebrew and schema
portions are committed. Book-derived rules data lands in a gitignored path (§14).

### 8.2 Characters

The sheet is stored as the tuple `(seed, ordered decision list, derived
snapshot)`. The snapshot is a cache; the seed and decisions are the truth, which
is what makes §2.2 real and lets a rules fix be replayed against existing
characters instead of invalidating them.

### 8.3 Retrieval corpus

Chunks and embeddings. Rebuildable from the PDFs, never committed, never backed
up anywhere off-machine.

### 8.4 The journal

Append-only, one row per mechanical event: rule id, inputs, RNG draw, modifiers,
result, timestamp. It serves as the audit trail for §2.2, the input to "what did
my change invalidate" in §4.3, and the debugging record when a player insists the
math is wrong. Treat it as a durable artifact, not a log.

---

## 9. Retrieval

### 9.1 What retrieval is for

Answering "what does the book say?" in the model's own narration. It is **never**
the path by which a mechanical value reaches the sheet — that is always §4. A
retrieved passage informs prose; it does not set a number.

### 9.2 Design

Hybrid search: vector similarity plus keyword. Rifts vocabulary is full of exact
tokens (`M.D.C.`, `P.P.E.`, specific O.C.C. names) where pure embedding search
underperforms and lexical matching is decisive.

Chunking follows the segmentation from §7.2 — an O.C.C. description is one
retrievable unit, because a chunk boundary through the middle of one produces
confidently wrong answers.

Every retrieved passage carries book and page, surfaced to the player so they can
check the real book. This is both a correctness feature and a §14 posture: the
system points at the source rather than substituting for it.

### 9.3 Embedding model coupling

The vector store is only valid for the embedding model that produced it. Record
the model identity alongside the vectors and refuse to query on mismatch. A
silent embedding-model swap degrades retrieval in a way that looks like a
prompting problem and wastes days.

---

## 10. API surface

REST plus SSE. Sketch, to be pinned when the backend exists.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/characters` | Start creation; optional explicit seed |
| `GET` | `/api/characters/{id}` | Sheet + pending choices + violations |
| `POST` | `/api/characters/{id}/turn` | Player utterance → SSE narration stream |
| `POST` | `/api/characters/{id}/choice` | Direct choice — the §5.4 form fallback |
| `POST` | `/api/characters/{id}/revise` | Return to an earlier state |
| `GET` | `/api/characters/{id}/journal` | Roll history (§8.4) |
| `GET` | `/api/rules/search` | Retrieval, with citations |
| `GET` | `/healthz` | Includes the §6.2 model reachability check |

The `choice` endpoint existing alongside `turn` is the API-level expression of
§5.4: every mechanical action is reachable without the model.

### 10.1 Streaming

Narration streams over SSE. Mechanical results are **not** streamed — they are
computed and persisted before narration begins (§5.1), then sent as one
structured event. The client must never render a number that arrived mid-stream
as though it were provisional.

---

## 11. Frontend

Next.js. Two surfaces: a conversation pane and a live character sheet that
updates from structured events rather than by parsing narration.

The sheet is the honest view of state. When narration and sheet disagree, the
sheet is right — and the disagreement is a bug in §5.1's ordering. Pending
choices render as real controls, which is how §5.4's fallback becomes free rather
than a separate build.

---

## 12. Secrets, security, and privacy

Single-player, local, no auth in v1. The threat model is not intrusion; it is
**accidental disclosure of licensed material**, which is §14.

- All configuration through environment variables; `.env` gitignored,
  `.env.example` committed with no real values.
- Required variables fail at startup rather than defaulting (§6.2).
- The model endpoint is the only network dependency in the serving path.

### 12.1 On the Drive credentials

`README.md`'s Quick Start mentions Drive credentials. That needs a decision
recorded here, because it is a data-flow question, not a config detail: **if
Google Drive is where the purchased PDFs live, then licensed material is
traversing a cloud account.**

That may be entirely fine — a private Drive holding books you own is no different
from a private disk. But it must be deliberate:

- Drive is a **source** for the offline pipeline only. Nothing in the serving
  path may hold Drive credentials.
- Sync is one-directional, into `./private/pdfs/`. The pipeline never writes
  back, so a derived corpus cannot escape upward.
- Scope the credential to read-only, and to one folder.

If Drive is not actually needed, remove it from the Quick Start — an unexplained
credential in setup instructions is worse than no credential.

---

## 13. Staged roadmap

One stage per card in `STATUS.md`. Each stage ends in something demonstrable;
none is "build the framework."

### Stage 0 — Repo hygiene ✅

`.gitignore` with the IP guard, `.env.example`, this document, `STATUS.md`.
**Ships before any pipeline code exists**, so there is never a window where
licensed material could be committed by accident.

The repo is **public**, which makes the remaining guard layers in §14.2 a
prerequisite for Stage 4 rather than a nicety — see §14.4.

### Stage 1 — Rules engine skeleton, no LLM

Dice with seeding, the eight attributes, the `Violation`/`Choice` types, and the
creation state machine with one O.C.C. hand-entered as homebrew-shaped data.
Tests only, no server. **Done when:** a legal character is generated from a seed
and a decision list, reproducibly, with zero model involvement.

Doing this first means §2.3 is true by construction rather than by discipline.

### Stage 2 — Backend and the form fallback

FastAPI over the engine, Postgres, the endpoints in §10 *except* `/turn`. Sheet
UI plus choice controls. **Done when:** a character can be created end-to-end
through the UI with no model running.

This is the stage most projects skip and then regret, because it is the only
version of the system whose correctness you can fully verify.

### Stage 3 — Model Runner integration

The client wrapper, the §6.2 health check, and the two-call turn loop with the
tool surface from §5.2. **Done when:** the same character can be created
conversationally, and killing the model mid-session degrades to Stage 2's form
rather than failing.

### Stage 4 — Content pipeline

Extraction through structured rules data for a single O.C.C., with provenance and
human review. **Done when:** one book-derived O.C.C. loads into the engine and
its tests pass — and `git status` is clean, proving the §14 guard works under
real conditions.

### Stage 5 — Retrieval

Chunk, embed, hybrid search, citations in narration. **Done when:** the model can
answer a rules question with a correct book-and-page citation.

### Stage 6 — Breadth

More O.C.C.s and R.C.C.s. Mostly pipeline and data work, and the first point at
which §2.5 pays for itself.

### Stage 7 — Play

Combat resolution, scenes, campaign state. Genuinely a new project; do not let
its design pull Stages 1–3 out of shape.

---

## 14. Content and IP

Rifts and the Palladium Megaversal system are copyrighted by Palladium Books.
This project is for personal use with material already purchased.

### 14.1 The distinction that matters

The `README.md` claim — that no official text is committed — is necessary but not
the whole picture, and it is worth being precise about why:

- **Mechanical facts** (an attribute minimum, a dice expression) are facts about
  a system. Our schema and our expression of them are our own.
- **Prose, tables as composed, names, and setting text** are the copyrighted
  expression. A structured extraction of a stat block preserves that
  expression's selection and arrangement and should be treated as a derivative
  work, not as neutral data.

So the retrieval corpus (§7.1) and book-derived rules data (§8.1) are both
treated as licensed material: **local-only, never committed, never distributed.**
Only code, schemas, and original content are committed.

### 14.2 Mechanical enforcement

Policy fails to the extent it depends on remembering. Three layers, all of which
**must be in place before Stage 4** — the first stage where licensed material
exists on disk:

1. **`.gitignore`** — `private/`, `data/derived/`, `*.pdf`, corpus and vector
   artifacts. Shipped in Stage 0.
2. **Pre-commit hook** — reject staged PDFs, files under the derived paths, and
   files over a size threshold. Catches the `git add .` case, which is the
   realistic one.
3. **CI check** — the same assertions, so a bypassed hook is still caught.

The design goal is that forgetting is safe.

Because the repo is public (§14.4), layers 2 and 3 are load-bearing rather than
defence in depth. A mistaken commit here is not a local problem to be quietly
amended away: it is published the moment it is pushed, and a force-push does not
reliably unpublish it — the objects remain reachable through the API, and
anything that has been cloned or indexed is gone for good. Treat the gap between
now and Stage 4 as the window in which layers 2 and 3 must close.

### 14.3 Directory contract

```
./private/            # gitignored. PDFs and anything derived from them.
./data/derived/       # gitignored. Book-derived rules data.
./data/homebrew/      # COMMITTED. Original content.
./data/schema/        # COMMITTED. Schemas — our own work.
```

One rule: **if it came from a book, it lives under a gitignored path.** No
exceptions and no judgement calls at commit time.

### 14.4 Distribution

The repo is **public**. The published artifact is code plus schemas plus
homebrew — **never data**. A recipient supplies their own PDFs and runs their own
pipeline against them. That separation is the whole basis on which this remains
personal use rather than redistribution, and public visibility makes it the
project's single most important invariant rather than a private preference.

The history was audited before the repo was published: eight blobs across three
commits, all documentation, no PDFs, no derived paths, no credentials. It went
public clean.

What changes going forward is the cost of a mistake, not the rule. Previously a
bad commit was recoverable in private; now it is a publication. Hence §14.2's
insistence that layers 2 and 3 land before any licensed material touches disk.

A useful test for any future change: **if someone clones this repo, they must not
be able to play Rifts with it.** If a change would make that false, the change is
wrong regardless of how convenient it is.

### 14.5 Licensing

The repo has **no license**, so it is all-rights-reserved by default. That was
the correct posture while private; now that it is public it is a holding
position, and one worth resolving deliberately — public visibility grants no
reuse rights, so the current state means anyone can read the code but nobody may
legally use it.

Whatever is chosen applies to **code, schemas, and homebrew only.** No license
this project grants can convey any right to Palladium's material, and the
LICENSE file should say so explicitly to avoid implying otherwise. Choosing one
is tracked in `STATUS.md`.
