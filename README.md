# ClaudeRPG

An LLM-driven, character-creation-first game set in Palladium's Rifts universe,
built from purchased Rifts PDF material. Runs locally on Docker Model Runner,
containerized, single-player to start.

## Status

**Design stage — no code yet.** The design is settled and written down; the
implementation has not started.

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — full design, invariants, and the
  staged roadmap (section 13)
- [`STATUS.md`](./STATUS.md) — stage tracking and open decisions

The repo is **public**, and contains no Palladium-owned material. See
**Content & IP** below for how that stays true.

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

This repo is **public**, so that rule is enforced mechanically rather than by
remembering: `.gitignore` today, plus a pre-commit hook and a CI check that must
land before the content pipeline is built. Anything that came from a book lives
under a gitignored path — no exceptions, no judgement calls at commit time.

Note that a structured extraction of a stat block is still a derivative of the
book's expression, so book-derived *rules data* is treated as licensed material
too, not just the prose. See [`ARCHITECTURE.md`](./ARCHITECTURE.md) section 14.

**If you clone this, you get code, schemas, and homebrew only.** There is no
Rifts content here and there never will be. Bring your own purchased PDFs and run
your own pipeline; the repo is useless for playing Rifts without the books, which
is deliberate.

## License

**Undecided — currently all rights reserved by default.** Public visibility is
not a license: without one, nobody may reuse this code, which is fine as a
holding position but worth resolving now that the repo is visible. Whatever is
chosen applies to code, schemas, and homebrew only — never to data. See
[`ARCHITECTURE.md`](./ARCHITECTURE.md) section 14.5.
