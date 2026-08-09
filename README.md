# Rifts LLM Game (working title)

An LLM-driven, character-creation-first game set in Palladium's Rifts universe,
built from purchased Rifts PDF material. Runs locally on Docker Model Runner,
containerized, single-player to start.

> Working title — rename freely once you land on something you like.

## Status

Early planning/scaffolding stage. See [`ARCHITECTURE.md`](./ARCHITECTURE.md)
for the full design and the staged roadmap (section 13). Track progress with a
`STATUS.md` or a GitHub Projects board, one card per stage.

## Prerequisites

- Docker Desktop (macOS, Apple Silicon) with Model Runner enabled
- Node.js (frontend)
- Python 3.11+ (backend)
- Your own Rifts PDF material (not included — this repo contains no
  Palladium-owned content, see **Content & IP** below)

## Quick Start

```bash
# clone the repo, then:
cp .env.example .env       # fill in DB creds, Drive credentials, etc.
docker compose up --build
```

Frontend: http://localhost:3000
Backend API: http://localhost:8000

Note: on macOS, Docker Model Runner executes chat/embedding/image models as a
host-native process (Metal GPU access), not inside a container. This is
expected — see `ARCHITECTURE.md` section 6.

## Project Structure (planned)

```
.
├── frontend/          # React/Next.js client
├── backend/           # FastAPI app (rules engine, orchestration, API)
├── content-pipeline/  # PDF ingestion → structured rules + embeddings
├── docker-compose.yml
├── ARCHITECTURE.md
└── STATUS.md          # stage tracking
```

## Content & IP

Rifts is copyrighted Palladium Books IP. This project is for personal use
against PDF material already purchased. No official Palladium text is
committed to this repo — only original/homebrew content and code. See
`ARCHITECTURE.md` section 14.

## License

TBD — pick a license once you decide whether/how this will ever be shared.
