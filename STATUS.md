# Status

Stage tracking for the roadmap in [`ARCHITECTURE.md`](./ARCHITECTURE.md) §13.
One row per stage. Keep the "done when" column honest — a stage is not complete
until its stated condition is demonstrably true.

Last updated: 2026-08-10

## Current stage: **Stage 1 — Rules engine skeleton**

Nothing is implemented yet. Stage 0 is complete; Stage 1 has not started.

| Stage | State | Done when |
|---|---|---|
| 0 — Repo hygiene | ✅ complete | IP guard in `.gitignore`, `.env.example`, `ARCHITECTURE.md`, `STATUS.md` |
| 1 — Rules engine skeleton (no LLM) | ⬜ not started | A legal character is generated from a seed + decision list, reproducibly, with no model running |
| 2 — Backend + form fallback | ⬜ not started | A character can be created end-to-end through the UI with no model running |
| 3 — Model Runner integration | ⬜ not started | Same character created conversationally; killing the model degrades to Stage 2 rather than failing |
| 4 — Content pipeline | ⬜ not started | One book-derived O.C.C. loads and passes tests, and `git status` is clean |
| 5 — Retrieval | ⬜ not started | Model answers a rules question with a correct book-and-page citation |
| 6 — Breadth | ⬜ not started | Additional O.C.C.s / R.C.C.s load as data with no code change |
| 7 — Play | ⬜ not started | Out of scope for now — treat as a separate project |

## Stage 0 — detail

- [x] `.gitignore` with IP containment rules (§14.2 layer 1)
- [x] `.env.example`
- [x] `ARCHITECTURE.md`
- [x] `STATUS.md`
- [x] Project renamed to **ClaudeRPG** — repo, local clone, README, and architecture doc now agree (2026-08-10)
- [x] History audited clean and repo made **public** (2026-08-10) — 8 blobs, 3 commits, all documentation
- [ ] **Pre-commit hook** rejecting PDFs / derived paths / oversized files (§14.2 layer 2)
- [ ] **CI check** duplicating the hook's assertions (§14.2 layer 3)
- [ ] License decision — currently unlicensed, so nobody may reuse the code (§14.5)

⚠️ **The two unchecked guard layers are now blocking for Stage 4.** The repo is
public, so a mistaken commit of licensed material is a publication, not a local
mistake — and a force-push does not reliably undo it. Nothing in Stage 4 starts
until layers 2 and 3 are in place. Stages 1–3 touch no licensed material and are
unaffected.

## Open decisions

Things `ARCHITECTURE.md` deliberately leaves unresolved. Record the answer in the
architecture doc when each is settled, not here.

| Decision | Where | Note |
|---|---|---|
| **Which license?** | §14.5 | Newly pressing — the repo is public but unlicensed, so nobody may reuse the code. Applies to code, schemas, and homebrew only; the LICENSE file must state that it conveys no rights to Palladium material. |
| Is Google Drive actually a PDF source? | §12.1 | If no, remove it from `README.md` and `.env.example`. An unexplained credential in setup is worse than none. |
| Postgres + pgvector, or SQLite + `sqlite-vec`? | §8 | Postgres assumed. SQLite is viable and drops a container; migration cost rises after Stage 2. |
| Chat model choice | §6.3 | Pick for tool-calling reliability over prose quality. |
| Embedding model choice | §6.3, §9.3 | Locked in once the corpus is built — changing it forces a full re-embed. |
| Cloud escalation for narration | §6.4 | Deferred. Note it is a §14 question first: escalating means sending licensed passages to a third party. |

## Settled

| Decision | When | Outcome |
|---|---|---|
| Repo visibility | 2026-08-10 | **Public.** Briefly private earlier the same day while the docs landed; published after a full-history audit found no licensed material (8 blobs, 3 commits, all documentation). Consequence: §14.2 layers 2 and 3 are now blocking for Stage 4 (§14.4). |
| Project name | 2026-08-10 | **ClaudeRPG.** Previously three identities — repo `RPG_SciFi`, README "Rifts LLM Game", description "SciFi based game". GitHub repo, local clone (`~/ClaudeRPG`), README, and `ARCHITECTURE.md` now all agree. |
