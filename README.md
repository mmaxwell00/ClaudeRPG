# Rifts LLM Game (working title)

An LLM-driven, character-creation-first game set in Palladium's Rifts universe,
built from purchased Rifts PDF material. Runs locally on Docker Model Runner,
containerized, single-player to start.

> Working title — rename freely once you land on something you like.

## Status

**Design stage — no code yet.** The design is settled and written down; the
implementation has not started.

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — full design, invariants, and the
  staged roadmap (section 13)
- [`STATUS.md`](./STATUS.md) — stage tracking and open decisions

The repo is **private**. See **Content & IP** below for why that matters.

## Prerequisites

- Docker Desktop (macOS, Apple Silicon) with Model Runner enabled
- Node.js (frontend)
- Python 3.11+ (backend)
- Your own Rifts PDF material (not included — this repo contains no
  Palladium-owned content, see **Content & IP** below)

## Quick Start

There isn't one yet — there is no `docker-compose.yml` and no application code.
Once Stage 2 lands ([`STATUS.md`](./STATUS.md)), it will be:

```bash
cp .env.example .env       # then fill it in — see the comments in that file
docker compose up --build
```

Frontend on `:3000`, backend API on `:8000`.

Note: on macOS, Docker Model Runner executes chat and embedding models as a
host-native process (for Metal GPU access), not inside a container. This is
expected, and it means the containerized backend has to reach *out* to the
host — the most likely thing to break during setup. See
[`ARCHITECTURE.md`](./ARCHITECTURE.md) section 6.2.

## Project Structure (planned)

Only the documentation and config files at the root exist today.

```
.
├── frontend/          # Next.js client
├── backend/           # FastAPI: rules engine, orchestration, API
├── content-pipeline/  # PDF ingestion → structured rules + embeddings (offline)
├── data/
│   ├── homebrew/      # committed — original content
│   ├── schema/        # committed — our own schemas
│   └── derived/       # GITIGNORED — book-derived rules data
├── private/           # GITIGNORED — your PDFs and anything derived from them
├── docker-compose.yml
├── ARCHITECTURE.md
└── STATUS.md
```

## Content & IP

Rifts and the Palladium Megaversal system are copyrighted by Palladium Books.
This project is for personal use with PDF material already purchased. Only code,
schemas, and original/homebrew content are committed.

Two things make that more than a promise:

1. The repo is **private**.
2. The rule is enforced mechanically, not by remembering — `.gitignore` now, plus
   a pre-commit hook and a CI check before the content pipeline is built. Anything
   that came from a book lives under a gitignored path, with no exceptions and no
   judgement calls at commit time.

Note that a structured extraction of a stat block is still a derivative of the
book's expression, so book-derived *rules data* is treated as licensed material
too — not just the prose. See [`ARCHITECTURE.md`](./ARCHITECTURE.md) section 14.

## License

None — all rights reserved by default, which is the right posture while the repo
is private. If this is ever shared, the shared artifact is code plus schemas plus
homebrew, never data: a recipient supplies their own PDFs. See section 14.5.
