# poker-arena

The production poker bot arena: a website and API where people register
poker bots, connect them from wherever they run, and play rated matches
against each other and against humans. Modelled closely on
[lichess](https://lichess.org) and its
[Bot API](https://lichess.org/api#tag/Bot).

**Public, like lila.** The arena's integrity doesn't depend on hiding its
code: secrets (the seed key, session keys, database credentials) exist
only in the deployment environment. Bot developers don't need this repo:
they work in
[poker-bot-template](https://github.com/FreddyyAndrews/poker-bot-template)
and talk to the arena only through its API.

**Status:** planning. Nothing is implemented yet.

---

## The three repos

| Repo | Role | lichess analogue | Visibility |
|---|---|---|---|
| [Poker-Harness](https://github.com/FreddyyAndrews/Poker-Harness) | Toolkit library: rules engine, bot protocol, local dev arena (matches, god view, probes, tests, comparisons, briefs), the bot bridge client and a mock server | python-chess (+ dev tools) | public |
| [poker-bot-template](https://github.com/FreddyyAndrews/poker-bot-template) | What a user forks and drops an agent into: a bot, dummy opponents, spot suites, bridge config, agent instructions | lichess-bot | public |
| poker-arena (this repo) | Production backend (FastAPI + Postgres) and frontend (React) | lila | public |

Both other repos depend on Poker-Harness, so the rules are identical in
development and production.

## How it works (the lichess model)

1. A user signs up on the arena website, registers a bot (a stable public
   name) and creates a token for it.
2. They fork poker-bot-template, put the token in `config.yml` and let an
   agent (for example in a Claude cloud session) develop the bot locally
   with full god-view tooling against dummy bots.
3. When ready, they run the bridge (`arena connect`) wherever they like.
   It opens the bot's event stream, accepts challenges or joins tables,
   and answers each decision by running their `bot.py` locally.
4. The arena never runs user code. It deals the cards (from secret seeds),
   sends each bot only what it may see, enforces time controls, records
   everything, and rates the results.
5. Anyone can watch matches live (public view) or play a bot. Owners query
   their own history (their cards plus showdowns) through the API, the
   `arena` CLI or MCP.

### Visibility rules

| Viewer | Live | After the match |
|---|---|---|
| A seated bot | its own cards (the `decide` state) | n/a |
| The bot's owner | its own cards | own cards, showdown cards, public actions, own bot's decisions and notes |
| Spectators | public actions, no hole cards | showdown cards and public actions |
| Admin (you) | god view | god view |

Opponents' notes, unrevealed cards and undealt board cards are never
exposed to bots or owners. The full deal is stored server-side only.

---

## Plan

### Backend (FastAPI + Postgres)

**A1. Skeleton**
- FastAPI app; SQLAlchemy 2 + Alembic; Postgres via docker compose; pytest
  with a throwaway database; CI.
- Poker-Harness as a pinned dependency (engine, protocol models, match
  runner).

**A2. Accounts and tokens**
- Sign up / log in (email + password first; GitHub OAuth later), web
  sessions.
- Bot registry: stable unique names, owner, profile, created/updated.
- Personal access tokens per bot, stored hashed, with scopes (`bot:play`,
  `bot:read`) and revocation. Several bots and tokens per account.

**A3. Bot API (lichess-style)**

Streams are newline-delimited JSON over long-lived HTTP responses;
actions are plain POSTs. The spec lives in Poker-Harness
(`docs/bot-api.md`), so the bridge, mock server and arena all follow one
document.
- `GET /api/bot/stream/event`: challenges, challenge cancellations,
  match start/finish. An open stream also means the bot is online.
- `POST /api/challenge/{bot}`, `.../{id}/accept|decline|cancel`:
  heads-up challenges with a format (blinds, stacks, hands, time control).
- `POST /api/seek` / `DELETE /api/seek/{id}`: join a queue for a table
  format (e.g. 6-max); tables start when full.
- `GET /api/bot/match/{id}/stream`: match setup, hand start (your cards
  only), public actions, `decide` requests (the same state shape as local
  development, plus a decision id and deadline), hand results with
  showdowns, match end.
- `POST /api/bot/match/{id}/decision`: `{decision_id, action, amount,
  logs}`.
- Time control: per-decision limit plus a time bank per match. On timeout
  or disconnect the seat checks if it can, otherwise folds; a match is
  abandoned for a bot after a reconnect grace period.

**A4. Match service**
- Runs matches with the Poker-Harness engine and match runner through a
  remote seat that bridges to the streams above. Many matches at once in
  one asyncio process.
- Hand seeds derived from the match seed with a server secret
  (HMAC-SHA256), never exposed.
- Stores the full event log server-side; every query goes through the
  visibility filter for the requesting viewer.
- Recovery: matches in progress when the server restarts are aborted
  cleanly and marked as such.

**A5. Owner API** (used by the CLI client and MCP)
- Own matches, hands and decisions (own perspective); own stats and
  briefs computed without hidden information; export a hand as a spot
  (opponents' unknown cards become random).

**A6. Public API**
- Bot profiles, online bots, leaderboards, live match feed (public view,
  pushed to the frontend over Server-Sent Events), finished match replays.

**A7. Matchmaking and ratings**
- Server-scheduled rated tables and events in addition to bot-driven
  challenges and seeks.
- Rating system: **to be decided** (Glicko-2 per format is the simple
  heads-up option; multi-seat tables need TrueSkill/OpenSkill or similar).

**A8. House bots**
- Reference bots run server-side (trusted code) so there is always
  someone to play.

**A9. Operations**
- Docker compose deployment on a VM with HTTPS (e.g. Caddy), backups,
  structured logs and metrics, rate limits per token and IP, abuse
  controls. Hosting is decided when the server is ready.

### Frontend (React + Vite + TypeScript)

- **F1. Account:** sign up / log in, register bots, create and revoke
  tokens (shown once).
- **F2. Browse:** bot profiles, leaderboards, match lists.
- **F3. Watch:** live table view (public), replay viewer with a scrubber
  that steps through every action, owner view of their own bots' matches,
  admin god view.
- **F4. Play:** a human takes a seat against bots (like challenging a bot
  on lichess).
- **F5. Admin:** users, bots, tokens, matches, abuse handling.

### Milestones (across the three repos)

1. **Protocol and local loop:** the bot API spec, bridge client and mock
   server in Poker-Harness; poker-bot-template connects to the mock
   server.
2. **Arena alpha:** A1-A5 + F1, run locally with docker compose; template
   bots play heads-up challenges against each other.
3. **Public beta:** A6-A9, F2-F3, deployed; the website links to the
   template.
4. **Play and polish:** F4-F5, ratings, scheduled events, MCP.

### Open decisions

- Rating system.
- Hosting (when the server is ready).
- Whether bots and owners see opponents' stable names (decided: yes) and
  how to handle collusion (low priority).
