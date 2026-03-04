# Real-Time Multiplayer Auction Platform Architecture (IPL Mock Auction Style)

This document proposes a production-ready architecture for a full-stack real-time multiplayer auction platform using:

- **Frontend:** React.js
- **Backend:** Node.js + Express
- **Realtime:** Socket.io
- **Database:** PostgreSQL (Supabase)
- **ORM:** Prisma (recommended; Sequelize optional)

---

## 1) Backend Architecture

## Services and Modules

Use a modular monolith first (easy to ship, easy to split later):

```txt
src/
  app.ts
  server.ts
  config/
  modules/
    auth/
    rooms/
    teams/
    players/
    auction/
    bids/
    chat/
    squad/
    export/
    history/
  realtime/
    socketServer.ts
    handlers/
    middlewares/
  common/
    errors/
    validators/
    guards/
    logger/
    events/
```

### Core Runtime Components
1. **REST API layer (Express):** room setup, room metadata, import, export requests, history.
2. **Socket layer (Socket.io):** live join flow, bidding, timers, activity feed, chat, admin controls.
3. **Auction Engine:** deterministic state machine + timer manager; validates all actions server-side.
4. **Persistence layer:** PostgreSQL with transactions and row locking for consistent bidding.
5. **Background workers (BullMQ/queues optional):** heavy exports (PDF/DOCX), large imports.

### Security Controls
- JWT/session auth for all users.
- Room-scoped access checks on every API/socket event.
- Team ownership binding at socket level: `socket.data = { userId, roomId, teamId, role }`.
- Strict RBAC: creator/admin-only actions validated in backend guards.
- Rate limiting on bid/chat events + anti-spam.
- Input validation (Zod/Joi) for every payload.

---

## 2) Database Schema (PostgreSQL)

> All domain tables include `room_id` to support isolated multi-room auctions.

```sql
-- users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  display_name TEXT NOT NULL,
  password_hash TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- rooms
CREATE TABLE rooms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_code TEXT UNIQUE NOT NULL,
  room_name TEXT NOT NULL,
  auction_name TEXT NOT NULL,
  creator_user_id UUID NOT NULL REFERENCES users(id),
  status TEXT NOT NULL DEFAULT 'LOBBY', -- LOBBY | LIVE | PAUSED | COMPLETED
  num_teams INT NOT NULL,
  initial_purse BIGINT NOT NULL,
  rtm_count INT NOT NULL,
  max_squad_size INT NOT NULL,
  min_squad_size INT NOT NULL,
  overseas_limit INT NOT NULL,
  bid_timer_seconds INT NOT NULL,
  chat_enabled BOOLEAN NOT NULL DEFAULT true,
  current_player_id UUID,
  current_bid BIGINT,
  current_bid_team_id UUID,
  version INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- teams
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  team_name TEXT NOT NULL,
  purse_remaining BIGINT NOT NULL,
  rtm_remaining INT NOT NULL,
  squad_count INT NOT NULL DEFAULT 0,
  overseas_count INT NOT NULL DEFAULT 0,
  controller_user_id UUID UNIQUE, -- one user controls one team in a room
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(room_id, team_name)
);

-- players
CREATE TABLE players (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  player_name TEXT NOT NULL,
  base_price BIGINT NOT NULL,
  role TEXT NOT NULL,
  nationality TEXT NOT NULL,
  is_overseas BOOLEAN NOT NULL DEFAULT false,
  previous_team TEXT,
  status TEXT NOT NULL DEFAULT 'UNSOLD', -- UNSOLD | IN_AUCTION | SOLD
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- bids
CREATE TABLE bids (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  player_id UUID NOT NULL REFERENCES players(id),
  team_id UUID NOT NULL REFERENCES teams(id),
  user_id UUID NOT NULL REFERENCES users(id),
  amount BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- squads
CREATE TABLE squads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  team_id UUID NOT NULL REFERENCES teams(id),
  player_id UUID NOT NULL REFERENCES players(id),
  buy_price BIGINT NOT NULL,
  via_rtm BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(room_id, player_id)
);

-- auction_history
CREATE TABLE auction_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  actor_user_id UUID REFERENCES users(id),
  event_type TEXT NOT NULL,
  event_data JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- optional: room participants
CREATE TABLE room_participants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  role TEXT NOT NULL DEFAULT 'PARTICIPANT', -- CREATOR | PARTICIPANT
  joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(room_id, user_id)
);

CREATE INDEX idx_teams_room_id ON teams(room_id);
CREATE INDEX idx_players_room_id ON players(room_id);
CREATE INDEX idx_bids_room_player ON bids(room_id, player_id);
CREATE INDEX idx_history_room_created ON auction_history(room_id, created_at);
```

---

## 3) Socket.io Design

## Room and Identity Binding
- On connect, authenticate token, load user.
- `join-room` event validates room code and membership.
- `select-team` event atomically assigns team controller if empty.
- Bind identity:

```ts
socket.data.userId = user.id;
socket.data.roomId = room.id;
socket.data.teamId = team.id;
socket.data.role = isCreator ? 'CREATOR' : 'PARTICIPANT';
socket.join(room.room_code);
```

## Event Contracts

### Client -> Server
- `room:join` `{ roomCode }`
- `team:select` `{ roomCode, teamId }`
- `auction:bid` `{ roomId, amount }`
- `auction:nextPlayer` `{ roomId }` (creator only)
- `auction:start|pause|resume|end` `{ roomId }` (creator only)
- `auction:changeTimer` `{ roomId, bidTimerSeconds }` (creator only)
- `chat:send` `{ roomId, message }`
- `chat:toggle` `{ roomId, enabled }` (creator only)

### Server -> Clients (room scoped via `io.to(roomCode).emit`)
- `room:state`
- `room:participants`
- `team:locked`
- `auction:started|paused|resumed|ended`
- `auction:playerChanged`
- `auction:bidAccepted`
- `auction:bidRejected`
- `auction:tick`
- `auction:playerSold`
- `chat:message`
- `chat:status`
- `activity:event`

---

## 4) Room Creation API

## `POST /api/rooms`
Creates a room + settings + initial teams/players.

**Payload**
```json
{
  "roomName": "Mega Auction Room",
  "auctionName": "IPL Mock Auction 2026",
  "numTeams": 10,
  "initialPurse": 100000000,
  "rtmCount": 2,
  "maxSquadSize": 25,
  "minSquadSize": 18,
  "overseasLimit": 8,
  "bidTimerSeconds": 20,
  "teams": [{ "teamName": "Mumbai" }],
  "players": [{ "playerName": "Player A", "basePrice": 2000000, "role": "BAT", "nationality": "India" }]
}
```

## SQL Import API
## `POST /api/rooms/:roomId/import-sql`
- Accept SQL insert snippets or uploaded CSV/JSON.
- Parse to safe internal DTOs (do **not** execute raw SQL directly).
- Validate and bulk insert in transaction.

---

## 5) Room Join + Team Selection Logic

1. User enters room code -> REST fetch room details and available teams.
2. User selects team -> socket emits `team:select`.
3. Server transaction:
   - lock team row (`SELECT ... FOR UPDATE`)
   - ensure `controller_user_id IS NULL`
   - assign `controller_user_id = current_user`
4. Save in socket context and acknowledge.
5. Reject duplicates, force single controller per team.

---

## 6) Auction Engine Logic

Implement as finite state machine:

- **States:** `LOBBY -> LIVE -> PAUSED -> LIVE -> COMPLETED`
- **Per-player states:** `IN_AUCTION -> SOLD/UNSOLD`

### Bid Validation (server-side only)
For each `auction:bid`:
1. Verify room state is `LIVE`.
2. Verify socket has bound `teamId` and belongs to room.
3. Verify bidder not current highest bidder (optional anti-self-bid).
4. Ensure `amount > currentBid` and follows increment policy.
5. Ensure `amount <= team.purse_remaining`.
6. Ensure `team.squad_count < max_squad_size`.
7. If player is overseas, ensure `team.overseas_count < overseas_limit`.
8. Write bid + update room current bid atomically.
9. Broadcast accepted bid and reset timer.

### Timer
- Per room timer manager in memory + DB checkpoint for recovery.
- Tick every second via `auction:tick`.
- On expiry with highest bid -> finalize sale transaction:
  - insert into `squads`
  - decrement purse
  - increment counts
  - player status `SOLD`
  - history event

---

## 7) Chat System

- Socket event `chat:send` stores messages (optional `chat_messages` table) and broadcasts.
- If `rooms.chat_enabled = false`, backend rejects messages.
- Creator uses `chat:toggle` to enable/disable.
- Apply profanity/spam filter and per-user throttling.

---

## 8) Admin Controls (Creator-Only)

Guard every control with `socket.data.role === 'CREATOR'` **and** DB verification of `rooms.creator_user_id`.

Actions:
- start, pause, resume, end auction
- change timer duration
- next player
- toggle chat
- edit configurable room settings (restricted during LIVE where needed)

All actions produce `auction_history` records.

---

## 9) Frontend React Component Structure

```txt
src/
  pages/
    CreateRoomPage.tsx
    JoinRoomPage.tsx
    TeamSelectionPage.tsx
    LobbyPage.tsx
    AuctionRoomPage.tsx
    HistoryPage.tsx
  components/
    room/RoomSettingsForm.tsx
    import/SqlImportPanel.tsx
    lobby/InviteCard.tsx
    lobby/TeamList.tsx
    lobby/ParticipantsList.tsx
    auction/CurrentPlayerCard.tsx
    auction/BidPanel.tsx
    auction/TimerDisplay.tsx
    auction/ActivityFeed.tsx
    auction/ChatPanel.tsx
    auction/SquadTab.tsx
    auction/PlayersTab.tsx
    auction/SettingsTab.tsx
    admin/AdminControls.tsx
  socket/
    socketClient.ts
    useRoomSocket.ts
  state/
    roomStore.ts
    auctionStore.ts
```

### Key UI Flows
- **Create Room:** settings + manual entry or SQL import panel.
- **Join Flow:** room code -> available teams -> team lock -> lobby.
- **Lobby:** invite link, chat, participants, teams, start button (creator only).
- **Auction Screen:** top player details + bid/timer + tabs (activity/chat/squad/community/settings).

---

## 10) Export PDF / Word Feature

## API
- `POST /api/rooms/:roomId/export` body `{ format: "pdf" | "docx" }`
- async job returns `jobId`; frontend polls `GET /api/exports/:jobId`

## Content in Export
- auction name + room metadata
- team-wise final squads
- each player buy price
- purse spent and remaining
- summary stats (sold/unsold players, highest bid, etc.)

## Libraries
- **PDF:** `pdfkit` or `puppeteer` (HTML template -> PDF)
- **DOCX:** `docx` npm package
- Save files in object storage (Supabase Storage / S3) and return signed URL.

---

## 11) Deployment Setup

## Environments
- **Frontend:** Vercel/Netlify
- **Backend + Socket:** Render/Fly.io/Railway/AWS ECS (single Node runtime)
- **Database:** Supabase PostgreSQL
- **Cache/Queue:** Redis (Upstash/Elasticache)

## Required Production Concerns
- Horizontal scaling with sticky sessions (or Socket.io Redis adapter).
- HTTPS + secure cookies/JWT rotation.
- DB migrations via Prisma Migrate.
- Observability: structured logs + metrics + error tracking (Sentry).
- Backups + point-in-time recovery.

## Socket Scaling
Use Redis adapter:

```ts
import { createAdapter } from '@socket.io/redis-adapter';
io.adapter(createAdapter(pubClient, subClient));
```

This keeps room broadcasts consistent across multiple backend instances.

---

## 12) Why This Is Scalable and Secure

- **Modular services** keep features isolated and maintainable.
- **Room-scoped data model** enables many independent concurrent auctions.
- **Server-authoritative bidding** prevents tampering and race-condition wins.
- **Transactional writes + locks** protect monetary and squad constraints.
- **Socket identity binding** enforces one-team-per-user bidding control.
- **Persistent auction history** supports auditability and result revisits.
