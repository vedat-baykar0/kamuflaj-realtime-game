# Kamuflaj

A real-time, browser-based social deduction game in the spirit of *Spyfall*. Three to
twelve players join a room from their own phones. Everyone is shown the same secret
word — except one player, the **Kamuflaj**, who only gets a vague hint and has to bluff
their way through. Players take turns submitting one associated word each, then vote on
who they think the impostor is.

I built this because I wanted a problem where the hard part wasn't the feature list, it
was the *state*. A dozen phones on flaky mobile networks, all needing to agree on whose
turn it is, who has voted, and who is allowed to know the secret — with no database to
fall back on and no room for a client to lie. That turned out to be a much better
teacher than another CRUD app.

> **Note on scope:** this repository is the application source, published for people who
> want to read the architecture. Deployment configuration and infrastructure scripts are
> intentionally not included.

---

## Table of contents

- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [The game state machine](#the-game-state-machine)
- [The WebSocket protocol](#the-websocket-protocol)
- [Design decisions worth explaining](#design-decisions-worth-explaining)
- [Local setup](#local-setup)
- [Project layout](#project-layout)
- [What I would do differently next](#what-i-would-do-differently-next)

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Backend | Python 3.12, FastAPI | Native `async`/`await` and first-class WebSocket support in one framework |
| Transport | WebSocket (`/ws`), JSON frames | Full-duplex; the server has to *push*, not wait to be polled |
| Server | Uvicorn (ASGI) | A single event loop handles every open socket without a thread per player |
| Templating | Jinja2 | One server-rendered shell; everything after that is driven over the socket |
| Frontend | Vanilla JavaScript, Tailwind CSS | No build step, no bundler, no framework runtime — see below |
| Persistence | None (in-process memory) | Deliberate; see [Design decisions](#design-decisions-worth-explaining) |

Three runtime dependencies. That is the entire `requirements.txt`.

## Architecture

The system is a single authoritative server with thin, stateless clients. Clients never
compute game state — they render whatever the server last told them.

```
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  Phone / Tab │   │  Phone / Tab │   │  Phone / Tab │   Vanilla JS client
  │  (player 1)  │   │  (player 2)  │   │  (player N)  │   renders state only
  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
         │                  │                  │
         │   WebSocket (ws:// upgraded to wss:// over TLS)
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │
                  ┌─────────▼──────────┐
                  │  FastAPI / Uvicorn │
                  │  single event loop │
                  └─────────┬──────────┘
                            │
          ┌─────────────────┼─────────────────────┐
          │                 │                     │
  ┌───────▼───────┐ ┌───────▼────────┐  ┌─────────▼────────┐
  │ Connection    │ │ Room registry  │  │ Background task  │
  │ registry      │ │ ROOMS: code →  │  │ prunes idle      │
  │ + rate limits │ │ Room(dataclass)│  │ rooms every 60 s │
  └───────────────┘ └───────┬────────┘  └──────────────────┘
                            │
                    ┌───────▼────────┐
                    │ asyncio.Lock   │  one lock per room:
                    │ per room       │  mutations are serialised
                    └────────────────┘
```

**The request cycle.** Every inbound frame follows the same path, and the path is the
whole design:

1. **Size and rate gate.** Frames over 4 KB are rejected. Each connection gets a sliding
   window of 40 messages per 5 seconds.
2. **Parse and dispatch.** `handle_message` resolves the frame's `type` against a command
   table. Four actions (`create_room`, `join_room`, `rejoin`, `ping`) are handled before
   session resolution; everything else requires an established session.
3. **Session resolution.** The room and player are looked up from the connection. A stale
   connection receives `session_invalid` and is told to rejoin rather than being silently
   dropped.
4. **Acquire the room lock.** `async with room.lock` — this is the part I had
   underestimated before building it. Two players can submit a word in the same
   event-loop tick; without the lock, the turn pointer advances twice and somebody loses
   their turn.
5. **Mutate, then broadcast.** The handler changes the `Room` dataclass, and then — this
   is the invariant the whole client depends on — `broadcast_state` sends a **complete
   state snapshot** to every connected player.

**Snapshots over diffs.** I send the full state after every mutation instead of
incremental patches. It costs more bytes, but it means a reconnecting client needs no
replay log and cannot drift out of sync. For a room capped at twelve players, bandwidth
was never the constraint; correctness was.

**Per-viewer redaction.** The snapshot is *built per recipient*, not broadcast verbatim.
This is the core security property of the game: the detectives' secret word is never
serialised into the impostor's payload, and the impostor's hint is never serialised into
anyone else's. The secret is not hidden with CSS or a client-side flag — it is absent
from the bytes on the wire. You cannot win this game with DevTools open.

## The game state machine

Each room is a finite state machine. Every transition is server-triggered — by a player
command, a host override, a disconnect, or a timer.

```
          create_room
               │
               ▼
          ┌─────────┐   start_game   ┌───────────┐
          │  LOBBY  │──────────────▶ │  PLAYING  │  turn-based word submission
          └─────────┘                └─────┬─────┘
               ▲                           │ word quota filled
               │                           ▼
               │                     ┌───────────┐
               │                     │  VOTING   │  one vote each, changeable
               │                     └─────┬─────┘
               │                           │ all votes in
               │                           ▼
               │                     ┌───────────┐
               │                     │ GUESSING  │  impostor gets 45 s, one guess
               │                     └─────┬─────┘
               │                           │ guess / skip / timeout
               │       play_again          ▼
               └───────────────────── ┌─────────┐
                                      │  ENDED  │  reveal + scoring
                                      └─────────┘

          TALKING — alternate branch for voice-chat mode:
          roles are dealt, discussion happens out loud,
          the host ends the round manually.
```

Phases are explicit named constants rather than booleans scattered across the room
object. Every command validates the current phase before it does anything, so a replayed
or hand-crafted frame — `vote` during the lobby, a second `guess` after time expired — is
rejected by the same check, not by luck.

Three game modes share this machine:

- **`online`** — everyone on their own device, words typed and broadcast.
- **`local`** — side-by-side voice play: the server only deals roles, discussion happens
  in the room, and the host reveals. Uses the `TALKING` branch.
- **`single`** — pass-the-phone. Runs entirely client-side and opens no room on the
  server; it only pulls a random word from `GET /api/word`.

## The WebSocket protocol

Every frame is a JSON object with a `type` discriminator.

**Client → server**

| Type | Purpose |
|---|---|
| `create_room` / `join_room` | Open a room, or join one by 4-character code |
| `rejoin` | Resume an existing session after a drop (see below) |
| `set_rounds` | Host sets words-per-player (1–4) |
| `start_game` | Host deals roles and starts the round |
| `submit_word` | Submit one word on your turn |
| `vote` | Vote for the suspected impostor; changeable until the tally closes |
| `guess` / `skip_guess` | Impostor's single attempt at the secret word |
| `start_voting` / `skip_turn` / `kick` / `end_round` | Host overrides for AFK players |
| `play_again` | Reset to lobby, carrying the scoreboard over |
| `leave` | Explicit exit |
| `ping` | Latency probe; answered with `pong` |

**Server → client**

| Type | Purpose |
|---|---|
| `state` | Full redacted snapshot — the client's single source of truth |
| `joined` | Session handshake: `player_id` + `token` |
| `error` | Machine-readable `code` plus a human-readable message |
| `kicked` / `room_closed` / `left` | Terminal events |
| `pong` | Latency response |

**Reconnection.** On join, the server issues a `player_id` and a
`secrets.token_urlsafe(16)` session token, which the client persists. If the socket dies —
backgrounded tab, tunnel, lost signal — the client reconnects and sends `rejoin`. The
server matches the token with `secrets.compare_digest` and restores the player to the same
seat, the same role, and the same score. Without the constant-time comparison, token
validation leaks timing information; with it, a player's identity cannot be hijacked by
someone who merely guesses a `player_id`.

**Failure handling is part of the protocol, not an afterthought.** A disconnected player's
turn is skipped automatically and excluded from the vote tally. If the impostor drops, the
guess phase is bypassed and the round resolves. If the host leaves, the room closes
cleanly with `room_closed` instead of stranding everyone in a dead lobby. If the headcount
falls below the minimum, the room returns to the lobby rather than deadlocking.

## Design decisions worth explaining

**No database.** Rooms are `@dataclass` objects in a dict, keyed by room code. A session
lasts minutes and nothing about it is worth surviving a restart, so a database would have
bought persistence I do not want, at the cost of latency I cannot spare and a schema I
would have to migrate. The trade-off is explicit: state is single-process, so the server
scales up, not out. Horizontal scaling would need a shared room registry — Redis pub/sub
is the obvious next step, and the `broadcast_state` boundary is deliberately where that
seam would go.

**Memory is bounded on purpose.** In-memory state plus public access is a leak waiting to
happen, so every structure has a ceiling: 4,000 rooms maximum, a 60-second pruner task
(started through FastAPI's `lifespan`) that reaps rooms idle for 15 minutes, a 60-entry cap
on each room's event log, and a rolling memory of the last 120 secret words so a group does
not see repeats.

**Vanilla JS, and I would choose it again.** A client whose only job is to render a
snapshot the server already computed does not need a reconciling virtual DOM. Dropping the
framework meant no build step, no `node_modules`, and a page that loads over bad mobile
data — which is exactly the network this game gets played on. The whole client is one
readable file of DOM rendering and socket handling.

**Validation lives on the server, always.** Word submissions must match
`^[^\W\d_]{2,24}$` (Unicode-aware: Turkish characters count as letters, digits and symbols
do not). Duplicate words within a round are rejected, and so is the secret word itself or
any part of it — that one matters, because a detective "accidentally" typing the secret
ends the game for everyone. Turkish text is case-folded through a custom map, since `İ`
and `ı` do not behave under ASCII `lower()`, so that `çaydanlık` and `caydanlik` compare
equal. The client mirrors some of these checks for instant feedback; the server treats
every one of them as untrusted anyway.

**Errors are codes, not strings.** Every failure returns a stable `code` alongside its
message, so the client can branch on behaviour — `rate_limited` throttles the UI,
`session_invalid` triggers a rejoin — without string-matching prose.

## Local setup

Requires Python 3.12 or newer.

```bash
git clone https://github.com/<your-username>/kamuflaj.git
cd kamuflaj

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Configuration is environment-driven. Copy the template and adjust if you need to:

```bash
cp .env.example .env
```

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | `8000` | Port to bind |
| `HOST` | `0.0.0.0` | Interface to bind (`127.0.0.1` for local-only) |
| `DEV` | unset | Set to any value to enable auto-reload |

Run it:

```bash
uvicorn main:app --reload --port 8000
```

Open `http://localhost:8000`. To test with real phones, bind `0.0.0.0` and visit
`http://<your-machine-ip>:8000` from a device on the same network — worth doing, since
mobile browsers are where the reconnection logic actually gets exercised.

### Docker

```bash
docker build -t kamuflaj .
docker run --rm -p 8000:8000 kamuflaj
```

### Verifying it is up

```bash
curl http://localhost:8000/healthz
# {"status":"ok","rooms":0,"players":0,"words":860,"layout":"nested"}
```

`GET /api/room/{code}` reports whether a room exists, its phase, and its headcount —
handy for debugging a session without attaching a second socket.

## Project layout

```
main.py                 FastAPI app, WebSocket protocol, room registry, game engine
words.py                Curated Turkish word pool with hints + random picker
templates/index.html    Server-rendered shell (Jinja2)
static/app.js           WebSocket client and all UI rendering
requirements.txt        Three runtime dependencies
Dockerfile              Container image
Procfile                Process definition for PaaS hosts
.env.example            Documented configuration template
```

## What I would do differently next

Writing this down because the limits are more interesting than the features:

- **Single-process state** is the real ceiling. Multi-instance deployment needs the room
  registry behind Redis, with `broadcast_state` publishing to a channel instead of
  iterating local connections.
- **No automated tests yet.** The state machine is the obvious target — `pytest` plus
  FastAPI's `TestClient` WebSocket support would let me assert transitions and the
  redaction invariant (no payload sent to the impostor ever contains the secret word),
  which is exactly the property a human tester will never check thoroughly enough.
- **Snapshot-per-mutation** is the right call at twelve players and the wrong one at
  twelve hundred. The migration path is diffing against a per-connection last-sent
  snapshot — same boundary, more bookkeeping.
- **Observability is thin.** `/healthz` reports counts; it does not report phase
  distribution, reconnection rates, or how often rooms die mid-round. Those are the
  numbers I would actually want.

---

## License

MIT — see [`LICENSE`](LICENSE).
