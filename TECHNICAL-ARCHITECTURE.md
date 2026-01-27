# Technical Architecture: Centrifugo + Redis + Backend Integration

This document provides in-depth technical details about how Centrifugo, Redis, and the Go backend work together to handle real-time gameplay and automatic disconnect detection.

---

## Table of Contents

1. [Overview](#overview)
2. [Centrifugo Architecture](#centrifugo-architecture)
3. [Redis Integration](#redis-integration)
4. [Backend Components](#backend-components)
5. [Data Flow](#data-flow)
6. [Disconnect Detection](#disconnect-detection)
7. [Channel Design](#channel-design)
8. [Security Model](#security-model)
9. [Performance Considerations](#performance-considerations)

---

## Overview

The system uses a **three-tier architecture** for real-time multiplayer gaming:

```
┌──────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                          │
│  Browser WebSocket Connections (centrifuge-js protocol)       │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         │ WebSocket (JWT Auth)
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                    CENTRIFUGO LAYER                           │
│  • WebSocket Hub (manages connections)                        │
│  • Channel Router (pub/sub distribution)                      │
│  • RPC Proxy (delegates to backend)                           │
│  • Presence Manager (tracks active users)                     │
└────────────────┬───────────────────┬─────────────────────────┘
                 │                   │
        Engine API (State)    RPC Calls (Game Logic)
                 │                   │
┌────────────────▼───────┐   ┌───────▼──────────────────────────┐
│     REDIS LAYER        │   │      BACKEND LAYER               │
│  • Broker (pub/sub)    │◄──┤  • Game State Manager            │
│  • Presence Storage    │   │  • RPC Handlers                  │
│  • History Buffer      │   │  • Disconnect Listener (pub/sub) │
│  • Stream Metadata     │   │  • Presence Monitor (polling)    │
└────────────────────────┘   └──────────────────────────────────┘
```

**Key Principle:** Centrifugo handles **transport and real-time distribution**; backend handles **game logic and state**; Redis provides **distributed state and messaging**.

---

## Centrifugo Architecture

### Engine Configuration

Centrifugo operates in **Redis Engine Mode** (vs. default Memory Engine):

```json
{
  "engine": "redis",
  "redis_address": "redis://pong-redis:6379/0",
  "redis_prefix": "centrifugo"
}
```

**What this enables:**
- Multi-node scaling (not used in this project, but architecture supports it)
- Persistent presence tracking
- Pub/sub distribution via Redis
- History storage in Redis streams

### Namespaces

Two namespaces partition channels by access control and feature set:

#### 1. `pong_public` Namespace
```json
{
  "name": "pong_public",
  "history_size": 10,
  "history_ttl": "300s",
  "force_recovery": true,
  "presence": true,
  "join_leave": true,
  "force_push_join_leave": true,
  "allow_subscribe_for_anonymous": true,
  "allow_subscribe_for_client": true,
  "allow_history_for_anonymous": true,
  "allow_history_for_client": true,
  "allow_publish_for_anonymous": false,
  "allow_presence_for_anonymous": false
}
```

**Purpose:** Broadcast channel for game events and spectators.

**Key Settings:**
- `history_size: 10` — Last 10 messages stored for late joiners/reconnects
- `force_recovery: true` — Clients recover missed messages on reconnect
- `presence: true` — Track who's subscribed (not used for game logic)
- `join_leave: true` — Emit join/leave events to all subscribers
- `force_push_join_leave: true` — Push events even if client didn't request

**Redis Keys Used:**
```
centrifugo.stream.pong_public:lobby          # Stream for history
centrifugo.stream.meta.pong_public:lobby     # Stream metadata
centrifugo.presence.data.pong_public:lobby   # Presence hash
centrifugo.presence.expire.pong_public:lobby # Presence expiration set
```

#### 2. `pong_private` Namespace
```json
{
  "name": "pong_private",
  "history_size": 0,
  "history_ttl": "0s",
  "force_recovery": false,
  "presence": true,
  "join_leave": true,
  "force_push_join_leave": true,
  "allow_subscribe_for_anonymous": false,
  "allow_publish_for_anonymous": false
}
```

**Purpose:** Point-to-point channels for enemy paddle updates.

**Key Settings:**
- `history_size: 0` — No history (real-time only, no replay)
- `presence: true` — **Critical for disconnect detection**
- `allow_subscribe_for_anonymous: false` — Requires JWT subscription token

**Why presence on private channels?**
Backend monitors `centrifugo.presence.data.pong_private:{first|second}` to detect when a player's WebSocket drops.

### RPC Proxy

Centrifugo delegates game logic to the backend via HTTP proxy:

```json
{
  "proxy_rpc_endpoint": "http://host.docker.internal:8080/v1/centrifugo/rpc",
  "proxy_include_connection_meta": true,
  "proxy_http_headers": ["X-Centrifugo-User-Id"]
}
```

**Flow:**
1. Client calls RPC: `centrifugo.rpc("pong.move", {dy: -5})`
2. Centrifugo forwards to backend with user context
3. Backend validates, updates state, publishes events
4. Centrifugo distributes events to subscribers

**Request Format (Centrifugo → Backend):**
```json
{
  "method": "pong.move",
  "data": "{\"dy\":-5,\"client_ts_ms\":1768567123}",
  "client": "b70458c7-4996-4e9f-896c-45914250cd50",
  "user": "test-token",
  "transport": "websocket",
  "protocol": "protobuf"
}
```

**Response Format (Backend → Centrifugo):**
```json
{
  "result": {
    "success": true,
    "new_y": 120,
    "server_ts_ms": 1768567124
  }
}
```

### JWT Authentication

Two token types secure access:

#### Connection Token (connect to Centrifugo)
```go
claims := jwt.MapClaims{
    "sub": userID,              // Subject (user identifier)
    "exp": now.Add(ttl).Unix(), // Expiration
    "iat": now.Unix(),          // Issued at
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
```

**Validated by:** Centrifugo using `token_hmac_secret_key`

#### Subscription Token (subscribe to private channels)
```go
claims := jwt.MapClaims{
    "sub":     userID,
    "channel": "pong_private:first",
    "exp":     now.Add(ttl).Unix(),
    "iat":     now.Unix(),
}
```

**Validated by:** Centrifugo before allowing subscription

---

## Redis Integration

### Data Structures Used

#### 1. **Presence Hash** (`centrifugo.presence.data.*`)
```redis
HGETALL centrifugo.presence.data.pong_private:first

# Returns:
"client_id_1" → '{"client":"abc123","user":"test-token","conn_info":{},"chan_info":{}}'
"client_id_2" → '{"client":"def456","user":"test-token","conn_info":{},"chan_info":{}}'
```

**Purpose:** Track active connections per channel.

**Backend Usage:**
- Presence monitor polls this hash every 2 seconds
- If hash is empty but player exists in lobby → trigger disconnect

**Expiration:** Hash keys expire when client disconnects (Centrifugo removes entry)

#### 2. **Pub/Sub Channels** (`centrifugo.*`)
```redis
PSUBSCRIBE centrifugo.*

# Messages received:
pmessage centrifugo.* centrifugo.control <control_message>
pmessage centrifugo.* centrifugo.node.xyz <node_message>
```

**Control Messages Format:**
```json
{
  "method": "leave",
  "params": {
    "channel": "pong_private:first",
    "info": {
      "user": "test-token",
      "client": "abc123"
    }
  }
}
```

**Backend Usage:**
- Subscribe to pattern `centrifugo.*`
- Filter for `method: "leave"` on `pong_private:*` channels
- Trigger disconnect handler for affected user

#### 3. **History Streams** (`centrifugo.stream.*`)
```redis
XRANGE centrifugo.stream.pong_public:lobby - +

# Returns:
1) "1768567123000-0" → '{"type":"player_joined","data":{"place":"first"}}'
2) "1768567124000-0" → '{"type":"game_started","data":{...}}'
```

**Purpose:** Store last N messages for recovery and late joiners.

**Managed by:** Centrifugo (backend never reads/writes directly)

#### 4. **Stream Metadata** (`centrifugo.stream.meta.*`)
```redis
HGETALL centrifugo.stream.meta.pong_public:lobby

# Returns:
"epoch" → "xyz"
"offset" → "1768567124000-0"
```

**Purpose:** Track stream position for recovery protocol.

### Pub/Sub vs. Presence: When Each is Used

| Detection Method | Trigger | Latency | Reliability | Use Case |
|------------------|---------|---------|-------------|----------|
| **Pub/Sub Listener** | `leave` event in `centrifugo.control` | ~10ms | High (clean disconnects) | Browser close, explicit leave |
| **Presence Monitor** | Empty hash for expected player | 0-2s | Very High (all cases) | Crashes, network failures, stale state |

**Why both?**
- Pub/sub catches **immediate, clean disconnects** (user closes tab)
- Presence catches **dirty disconnects** (network timeout, browser crash, Centrifugo restart)

---

## Backend Components

### 1. Disconnect Module

#### `redis_subscriber.go`

**Purpose:** Listen for real-time disconnect events via Redis pub/sub.

**Implementation:**
```go
type RedisSubscriber struct {
    client          *redis.Client
    pubsubPattern   string          // "centrifugo.*"
    disconnectFn    func(userID string)
    privatePrefix   string          // "pong_private:"
    reconnectWindow time.Duration   // 2s grace period
}

func (s *RedisSubscriber) Start(ctx context.Context) error {
    pubsub := s.client.PSubscribe(ctx, s.pubsubPattern)
    defer pubsub.Close()

    for {
        msg, err := pubsub.ReceiveMessage(ctx)
        if err != nil { /* handle */ }
        
        s.handleMessage(msg.Payload)
    }
}
```

**Message Handling:**
1. Parse JSON envelope: `{"method":"leave","params":{...}}`
2. Check if channel is `pong_private:*`
3. Extract user ID from `params.info.user`
4. Schedule disconnect after 2-second grace period (allows quick reconnects)

**Grace Period Logic:**
```go
time.AfterFunc(s.reconnectWindow, func() {
    s.disconnectFn(userID)
})
```

If user reconnects within 2 seconds, they still appear in presence hash when disconnect fires → no action taken.

#### `presence_monitor.go`

**Purpose:** Poll Redis presence hashes to detect stale lobby seats.

**Implementation:**
```go
type PresenceMonitor struct {
    client         *redis.Client
    redisPrefix    string       // "centrifugo"
    checkInterval  time.Duration // 2s
    disconnectFn   func(userID string)
}

func (m *PresenceMonitor) Start(ctx context.Context) error {
    ticker := time.NewTicker(m.checkInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            m.checkLobby()
        case <-ctx.Done():
            return nil
        }
    }
}
```

**Check Logic:**
```go
func (m *PresenceMonitor) checkLobby() {
    lobby := game.GlobalLobby.GetStatus()
    
    // Check first player
    if lobby.FirstConnected {
        if !m.isPresent("pong_private:first", lobby.FirstUserID) {
            m.disconnectFn(lobby.FirstUserID)
        }
    }
    
    // Check second player
    if lobby.SecondConnected {
        if !m.isPresent("pong_private:second", lobby.SecondUserID) {
            m.disconnectFn(lobby.SecondUserID)
        }
    }
}

func (m *PresenceMonitor) isPresent(channel, userID string) bool {
    key := fmt.Sprintf("%s.presence.data.%s", m.redisPrefix, channel)
    entries := m.client.HGetAll(ctx, key)
    
    // Check if any entry has matching user
    for _, value := range entries {
        var info struct{ User string `json:"user"` }
        json.Unmarshal([]byte(value), &info)
        if info.User == userID {
            return true
        }
    }
    return false
}
```

**Edge Cases Handled:**
- Player joins but never connects WS → seat cleared after 2s
- Player refreshes browser → temporary absence detected, game ends
- Network hiccup → if reconnect < 2s, no action; if > 2s, game ends

### 2. Game State Manager

**Global Lobby:** Single-instance, mutex-protected state.

```go
type Lobby struct {
    mu sync.RWMutex
    
    FirstPlayer  *Player
    SecondPlayer *Player
    
    FirstPaddleY  int
    SecondPaddleY int
    
    FirstScore  int
    SecondScore int
    
    GameStarted  bool
    LastGoalTime time.Time
    UpdatedAt    time.Time
}
```

**Thread Safety:** All methods use `mu.Lock()` or `mu.RLock()`.

**Key Operations:**

```go
// Join: Add player, start game if both present
func (l *Lobby) Join(userID string, place Place) error

// Leave: Remove player, return if game was active
func (l *Lobby) Leave(userID string) (Place, bool, error)

// GetPlayerPlace: Check which seat user occupies
func (l *Lobby) GetPlayerPlace(userID string) (Place, error)

// Reset: Clear all state (called after game end)
func (l *Lobby) Reset()
```

### 3. RPC Handlers

#### `pong.move`
```go
func (h *Handler) handleMove(userID string, params MoveParams) {
    // 1. Validate user is in game
    place := game.GlobalLobby.GetPlayerPlace(userID)
    
    // 2. Validate dy within bounds (-20 to +20)
    if params.DY < -20 || params.DY > 20 {
        return error
    }
    
    // 3. Update paddle position
    newY := game.GlobalLobby.UpdatePaddle(place, params.DY)
    
    // 4. Publish to opponent's private channel
    opponentChannel := game.GetOpponentChannel(place)
    centClient.PublishEnemyMove(opponentChannel, newY)
    
    // 5. Publish to public channel (spectators + both players)
    centClient.PublishPublicEvent("pong_public:lobby", "move", {
        place: place,
        paddle_y: newY,
        ball_x: params.BallX,  // Optional, forwarded from first player
        ball_y: params.BallY,
    })
}
```

**Why publish twice?**
- **Private channel:** Low-latency paddle position for opponent (no history, no spectators)
- **Public channel:** Full game state with history for late joiners/spectators

#### `pong.goal`
```go
func (h *Handler) handleGoal(userID string, params GoalParams) {
    // 1. Verify user is in active game
    
    // 2. Add goal with duplicate prevention (500ms window)
    firstScore, secondScore, gameWon, winner := 
        game.GlobalLobby.AddGoal(params.ScoredBy)
    
    // 3. Publish goal event
    centClient.PublishPublicEvent("pong_public:lobby", "goal", {
        scored_by: params.ScoredBy,
        first_score: firstScore,
        second_score: secondScore,
    })
    
    // 4. If someone won (score >= 10), end game
    if gameWon {
        h.endGame(ctx, "win", winner)
    }
}
```

**Duplicate Goal Prevention:**
```go
if now.Sub(l.LastGoalTime) < 500*time.Millisecond {
    return currentScores, false, ""
}
l.LastGoalTime = now
```

Prevents both players reporting the same goal due to network race conditions.

---

## Data Flow

### 1. Player Join Flow

```
1. Browser → Backend: POST /v1/games/join {"place":"first"}
   ├─ Backend validates seat availability
   ├─ Adds player to lobby
   └─ Generates JWT tokens
       ├─ Connection token (sub: userID)
       └─ Subscription token (sub: userID, channel: pong_private:first)

2. Browser → Centrifugo: Connect with connection_token
   ├─ Centrifugo validates JWT
   ├─ Establishes WebSocket
   └─ Returns client ID

3. Browser → Centrifugo: Subscribe pong_public:lobby
   ├─ Centrifugo allows (anonymous OK)
   ├─ Sends last 10 history messages
   └─ Client recovers game state

4. Browser → Centrifugo: Subscribe pong_private:first with subscribe_token
   ├─ Centrifugo validates subscription JWT
   ├─ Adds to presence hash
   │   → Redis: HSET centrifugo.presence.data.pong_private:first
   ├─ Publishes join event (if join_leave enabled)
   │   → Redis: PUBLISH centrifugo.control {"method":"join",...}
   └─ Client ready to receive enemy moves

5. Backend publishes player_joined event
   ├─ Backend → Centrifugo API: POST /api/publish
   │   {
   │     "channel": "pong_public:lobby",
   │     "data": {"type":"player_joined","data":{...}}
   │   }
   └─ Centrifugo → All subscribers in pong_public:lobby
```

**Redis State After Join:**
```
centrifugo.presence.data.pong_private:first = {
    "abc123": '{"user":"test-token","client":"abc123",...}'
}
centrifugo.stream.pong_public:lobby = [
    {...},
    {"type":"player_joined","data":{"place":"first"}}
]
```

### 2. Game Loop (Paddle Movement)

```
Player 1 moves paddle:

1. Browser (P1) detects arrow key
   └─ Calculates dy = -5 (moving up)

2. Browser (P1) → Centrifugo RPC: pong.move {dy: -5}
   └─ Centrifugo → Backend: POST /v1/centrifugo/rpc
       {
         "method": "pong.move",
         "user": "player1-token",
         "data": "{\"dy\":-5}"
       }

3. Backend processes move
   ├─ Validates dy in range (-20, 20)
   ├─ Updates FirstPaddleY: 115 → 110
   ├─ Publishes to pong_private:second
   │   Backend → Centrifugo API: POST /api/publish
   │   {
   │     "channel": "pong_private:second",
   │     "data": {"type":"enemy_move","enemy_paddle_y":110}
   │   }
   └─ Publishes to pong_public:lobby
       {
         "channel": "pong_public:lobby",
         "data": {"type":"move","place":"first","paddle_y":110}
       }

4. Centrifugo distributes
   ├─ pong_private:second → Player 2 WebSocket
   │   (Only Player 2 receives this, low latency)
   └─ pong_public:lobby → All subscribers
       ├─ Player 1 (sees own move confirmed)
       ├─ Player 2 (redundant, already got from private)
       └─ Spectators (see game update)

5. Browser (P2) receives enemy_move
   └─ Updates opponent paddle position in canvas
```

**Latency Optimization:**
- Private channel: Backend → Centrifugo → Player2 ≈ 5-15ms
- Client-side prediction: Player1 updates locally immediately, backend confirms

### 3. Disconnect Flow (Browser Close)

```
Player 1 closes browser:

1. Browser → Centrifugo: WebSocket connection closed
   (TCP FIN or timeout)

2. Centrifugo detects disconnect
   ├─ Removes from all subscriptions
   ├─ Deletes from presence hash
   │   → Redis: HDEL centrifugo.presence.data.pong_private:first abc123
   └─ Publishes leave event
       → Redis: PUBLISH centrifugo.control
         {
           "method": "leave",
           "params": {
             "channel": "pong_private:first",
             "info": {"user": "player1-token", "client": "abc123"}
           }
         }

3A. Backend Redis Subscriber (fast path, ~10ms)
   ├─ Receives leave message
   ├─ Filters: channel matches "pong_private:*"
   ├─ Extracts userID: "player1-token"
   └─ Schedules disconnect after 2s grace period
       time.AfterFunc(2s, func() {
           handler.HandleDisconnect("player1-token")
       })

3B. Backend Presence Monitor (slow path, 0-2s)
   ├─ Next tick (within 2s)
   ├─ Checks lobby.FirstUserID = "player1-token"
   ├─ Queries Redis: HGETALL centrifugo.presence.data.pong_private:first
   ├─ Result: empty hash (no entries)
   └─ Calls handler.HandleDisconnect("player1-token")

4. Backend HandleDisconnect
   ├─ Checks if user still in lobby (might have already left)
   ├─ Calls game.GlobalLobby.Leave("player1-token")
   │   → Returns: place="first", wasGameActive=true
   ├─ Publishes player_left event
   │   Centrifugo API → pong_public:lobby
   └─ Calls endGame("player_left", "")
       ├─ Publishes game_ended event
       ├─ Clears history: DELETE centrifugo.stream.pong_public:lobby
       └─ Resets lobby state

5. Centrifugo → Remaining players
   ├─ player_left event
   └─ game_ended event

6. Browser (P2) receives game_ended
   └─ Shows "Game Over: Player Left" message
```

**Why 2-second grace period?**
- Page refresh: disconnect → reconnect in ~500ms
- Network hiccup: temporary drop, auto-reconnect
- Without grace: game ends on every page refresh (bad UX)

---

## Disconnect Detection

### Dual-Strategy Approach

| Strategy | Mechanism | Latency | Handles | Misses |
|----------|-----------|---------|---------|--------|
| **Pub/Sub** | `centrifugo.control` leave events | 10-50ms | Clean disconnects | Crashes, Redis failures |
| **Presence Polling** | Redis hash checks every 2s | 0-2s | All cases | Nothing (eventual consistency) |

**Combined:** Pub/sub gives fast response for common cases; presence ensures no user is "stuck" in lobby.

### Edge Cases Matrix

| Scenario | Pub/Sub | Presence | Outcome |
|----------|---------|----------|---------|
| User closes tab | ✅ Leave event | ✅ Hash cleared | Game ends in ~2s |
| Browser crash | ❌ No event | ✅ Hash cleared after timeout | Game ends in 2-4s |
| Network timeout | ❌ Delayed/lost | ✅ Hash cleared | Game ends in 2-4s |
| Centrifugo restart | ❌ No events | ✅ Hash cleared on restart | Game ends in 2-4s |
| Redis restart | ❌ Events lost | ❌ Hash lost, then recreated | Seats cleared, players can rejoin |
| Backend restart | ⚠️ Events missed during downtime | ✅ First poll after restart | Seats cleared on first check |
| Page refresh (fast) | ✅ Leave + Join | ✅ Hash updated | No game end (reconnect < 2s) |
| Page refresh (slow) | ✅ Leave event | ✅ Hash cleared | Game ends (reconnect > 2s) |

### Reconnect Grace Period

```go
// redis_subscriber.go
time.AfterFunc(s.reconnectWindow, func() {
    s.disconnectFn(userID)
})
```

**Why delay?**
1. User refreshes page: disconnect → immediate reconnect
2. Network switch (WiFi → 4G): brief gap, auto-reconnect
3. Centrifugo node switch (multi-node): seamless handoff

**Alternative considered:** Immediate disconnect
- ❌ Every page refresh ends game
- ❌ Mobile users switching networks = game over
- ❌ Poor UX for unstable connections

---

## Channel Design

### Channel Hierarchy

```
pong_public:lobby          # 1:N broadcast (all clients)
    ├─ player_joined       # Event: new player
    ├─ player_left         # Event: player quit
    ├─ game_started        # Event: both ready
    ├─ move                # Event: paddle + ball state
    ├─ goal                # Event: score update
    └─ game_ended          # Event: game over

pong_private:first         # 1:1 direct (second → first)
    └─ enemy_move          # Message: opponent paddle

pong_private:second        # 1:1 direct (first → second)
    └─ enemy_move          # Message: opponent paddle
```

### Why Private Channels?

**Problem:** If only public channel existed:
```
Player 1 moves → Backend → pong_public:lobby → ALL subscribers
    ├─ Player 1 (redundant, client predicted)
    ├─ Player 2 (needed)
    └─ 50 spectators (needed)
```

**Solution:** Split into public (everyone) + private (opponent only):
```
Player 1 moves → Backend → TWO publishes:
    ├─ pong_private:second → Player 2 ONLY (low latency)
    └─ pong_public:lobby → Everyone (with history)
```

**Benefits:**
- Lower latency for opponent (smaller message, fewer recipients)
- Bandwidth savings (opponent doesn't receive redundant public message)
- Presence tracking per player (detect individual disconnects)

### Message Deduplication

**Problem:** Player receives move via both channels.

**Solution:** Frontend filters:
```javascript
// centrifugo.js
if (event.type === 'enemy_move') {
    // Only from private channel, update opponent
    updateEnemyPaddle(event.enemy_paddle_y);
} else if (event.type === 'move' && event.place !== myPlace) {
    // From public, but ignore if it's opponent (already got from private)
    if (!isPlayerActive) {  // I'm spectator
        updatePaddle(event.place, event.paddle_y);
    }
}
```

---

## Security Model

### Authentication Layers

1. **HTTP API:** `Authorization: Bearer <token>` + `X-User-ID: <id>`
   - Simple middleware, not crypto-verified
   - Acceptable for demo (would use real auth in production)

2. **WebSocket Connection:** JWT connection token
   - Centrifugo validates `sub` claim
   - Prevents unauthorized WS connections

3. **Private Channel Subscription:** JWT subscription token
   - Centrifugo validates `sub` + `channel` claims
   - Prevents player1 from subscribing to player2's channel

### Token Flow

```
Join Request:
    Backend generates:
        connection_token = JWT{sub: userID, exp: now+15m}
        subscribe_token = JWT{sub: userID, channel: pong_private:first, exp: now+15m}
    
    Returns to client:
        {
            connection_token: "eyJ...",
            subscribe_token: "eyJ...",
            private_channel: "pong_private:first"
        }

Client connects:
    centrifugo.setToken(connection_token)
    centrifugo.connect()
    
    centrifugo.subscribe(private_channel, subscribe_token)
```

### HMAC Secret Sharing

**Critical:** Backend and Centrifugo must share same secret:
```
iac/centrifugo/config.json:
    "token_hmac_secret_key": "my-secret-key-123"

pong-backend/.env:
    CENTRIFUGO_SECRET=my-secret-key-123
```

**If mismatch:**
- Backend generates tokens with secret A
- Centrifugo validates with secret B
- All connections rejected (signature mismatch)

---

## Performance Considerations

### Scalability Limits (Current Architecture)

| Component | Limit | Bottleneck | Solution if Exceeded |
|-----------|-------|------------|---------------------|
| **Centrifugo** | ~10K connections | CPU (WebSocket framing) | Add nodes behind load balancer |
| **Redis** | ~50K ops/sec | Network bandwidth | Redis Cluster sharding |
| **Backend** | ~1K RPC/sec | Single Go process | Horizontal scaling (stateless) |
| **Game Lobby** | 2 players | Global singleton | Multi-lobby support |

### Message Size Optimization

**Typical message sizes:**
```
player_joined: ~80 bytes
move: ~120 bytes (with ball state)
goal: ~90 bytes
enemy_move: ~50 bytes
```

**Optimization:**
- Binary protocol (protobuf) available but not used (JSON is human-readable for debugging)
- Compression not enabled (messages too small to benefit)

### Redis Key Expiration

**Presence keys:** Auto-expire when client disconnects (Centrifugo manages)

**History keys:** TTL = 300s (5 minutes)
```
centrifugo.stream.pong_public:lobby → expires after 5min of inactivity
```

**Why 5 minutes?**
- Spectators can view recent games
- Old games don't accumulate
- Balance between history depth and memory

### Connection Pooling

**Backend → Redis:**
```go
redis.NewClient(&redis.Options{
    PoolSize: 10,  // Default
    MinIdleConns: 2,
})
```

**Backend → Centrifugo HTTP API:**
```go
gocent.New(gocent.Config{
    // Uses Go's default http.Client (connection pooling enabled)
})
```

---

## Debugging Tips

### Monitor Redis Pub/Sub
```bash
docker exec pong-redis redis-cli PSUBSCRIBE "centrifugo.*"
```

### Check Presence
```bash
docker exec pong-redis redis-cli HGETALL centrifugo.presence.data.pong_private:first
```

### View History
```bash
docker exec pong-redis redis-cli XRANGE centrifugo.stream.pong_public:lobby - +
```

### Backend Logs
```bash
docker logs pong-api -f | grep -E "(disconnect|left|ended)"
```

### Centrifugo Admin Panel
```
http://localhost:8000/
Username: admin
Password: (from config.json)
```

---

## Future Improvements

1. **Multi-lobby support:** Replace global singleton with lobby manager
2. **Ranked matchmaking:** Queue system, ELO ratings
3. **Replay system:** Store full game in Redis streams, playback later
4. **Anti-cheat:** Server-authoritative ball physics
5. **Metrics:** Prometheus exporter for connection counts, latency
6. **Graceful shutdown:** Drain connections before restart

---

**Last Updated:** 2026-01-16
