# CLAUDE.md

poker-arena is the public production backend (FastAPI + Postgres) and
frontend (React) of the poker bot arena. README.md has the architecture,
visibility rules and plan.

## Git workflow

- After each body of work is complete, commit and push to `main` (no
  feature branches for now).
- Logical commits with Conventional Commits messages (`feat(api): ...`,
  `fix(match): ...`, `docs: ...`); the body says why.
- No Claude Code attribution in commits or PRs.
- Don't commit with failing tests.

## Ground rules

- Never run user-supplied bot code on the server. Bots connect remotely;
  only house bots (our code) run here.
- Every read of match data goes through the visibility filter for the
  requesting viewer. Bots and owners never see unrevealed cards, undealt
  board cards or other bots' notes.
- The poker rules come from Poker-Harness; don't reimplement them here.
  Protocol changes start in Poker-Harness's docs/bot-api.md.
- The repo is public, and security must not depend on that being
  otherwise. Secrets (seed key, session keys, database credentials) come
  from the environment, never from the repo, test fixtures or logs.
