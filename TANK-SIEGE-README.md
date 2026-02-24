# TANK SIEGE — Complete Browser Tank Battle Game

## Quick Start

**Local multiplayer:** Open `tank-siege.html` in Chrome or Firefox. Click "Local Multiplayer", set player count (2–8), customize key bindings, click "Start Game."

**Online multiplayer:**
1. **Host:** Open the file, click "Online Multiplayer" → "Create Lobby". Share the 6-character join code with friends.
2. **Clients:** Open the file on separate machines, click "Online Multiplayer" → enter your name + the join code → "Join Lobby" → click Ready.
3. Host clicks "Start Game" once everyone is ready.

> **Requires internet** for the WebRTC signaling handshake (PeerJS cloud broker: `0.peerjs.com`). After the connection is established, all game data flows directly P2P with no intermediary server.

---

## Networking Architecture

### Overview
```
CLIENT A ──[WebRTC DataChannel]──┐
CLIENT B ──[WebRTC DataChannel]──┤  HOST (Authoritative Simulation)
CLIENT C ──[WebRTC DataChannel]──┘
                  │
         [PeerJS Broker ONLY for]
         [SDP offer/answer + ICE]
         [No game data ever sent]
```

### Signaling (WebRTC Handshake Only)
- Uses **PeerJS cloud broker** (`0.peerjs.com:443`) solely for WebRTC signaling
- Host registers as peer ID `"TS_<LOBBYCODE>"` (e.g., `TS_XKQR72`)
- Clients connect by peer ID derived from the join code
- After ICE negotiation completes, the DataChannel is direct P2P
- **Zero game data ever touches the signaling server**

### Host-Authoritative Model
```
HOST:
  ┌─────────────────────────────────────────┐
  │  Fixed 60 Hz Simulation Loop            │
  │  ┌──────────────────────────────────┐   │
  │  │ Collect inputs (all players)     │   │
  │  │ Advance simulation 1 tick        │   │
  │  │ Resolve physics                  │   │
  │  │ Check round end                  │   │
  │  └──────────────────────────────────┘   │
  │  Every 3 ticks (~20 Hz):                │
  │    Serialize state → broadcast snapshot │
  └─────────────────────────────────────────┘

CLIENT:
  ┌─────────────────────────────────────────┐
  │  Read local keys → send INPUT packet    │
  │  Receive SNAPSHOT → applySnapshot()     │
  │  Renderer runs at display refresh rate  │
  └─────────────────────────────────────────┘
```

### Packet Types
| ID | Direction | Purpose |
|----|-----------|---------|
| 1  | Host→Client | Lobby state broadcast |
| 2  | Host→Client | Game start (seed, players, round) |
| 3  | Host→Client | Authoritative state snapshot |
| 4  | Host→Client | Round end + scores |
| 5  | Host→Client | Kick (lobby full, game in progress) |
| 6  | Host→Client | Pong (ping response) |
| 10 | Client→Host | Join request (name, ID) |
| 11 | Client→Host | Input state (per tick) |
| 12 | Client→Host | Ready toggle |
| 13 | Client→Host | Ping probe |

### Client-Side Prediction & Reconciliation
Clients shadow-run the simulation locally using received inputs and apply authoritative snapshots when they arrive. Since the host snapshot overwrites tank positions/states directly, clients stay synchronized within one snapshot interval (~50ms at 20 Hz) without any complex rollback implementation. This is appropriate because:
- The simulation is deterministic (fixed timestep, seeded RNG)
- Inputs only affect the client's own tank
- The host has final authority on all state

For a production game, full rollback + replay would be used, but this approach produces correct, smooth results for typical broadband connections.

---

## Code Architecture

The game is a single self-contained HTML file (~85KB) with no build tools required. The JavaScript is organized into labeled sections:

| Section | Class/Object | Responsibility |
|---------|-------------|----------------|
| 1 | `C` (Constants) | All tunable parameters |
| 2 | `Utils` | Encoding, ID gen, math helpers |
| 3 | `RNG` | Mulberry32 seeded PRNG for determinism |
| 4 | `Maze` | Iterative DFS maze generation |
| 5 | `Vec2` | 2D vector math |
| 6 | `Physics` | Collision detection & resolution |
| 7 | `Simulation` | Authoritative 60 Hz game loop |
| 8 | `Pkt` | Packet builders & validators |
| 9 | `WebRTCHost` | Host networking (PeerJS + DataChannels) |
| 10 | `WebRTCClient` | Client networking + prediction |
| 11 | `LocalGame` | Same-keyboard multiplayer |
| 12 | `Particles` | Visual effects |
| 13 | `Renderer` | Canvas 2D rendering (decoupled loop) |
| 14 | `startMenuBG` | Animated background |
| 15 | `KeyBindingsUI` | Key rebinding UI |
| 16 | `Screens` | Screen state manager |
| 17 | `App` | Main controller / event wiring |

---

## Physics Details

### Bullet Reflection
Implements the standard specular reflection formula:
```
R = V - 2(V·N)N
```
Where `V` = incoming velocity vector, `N` = wall surface normal (axis-aligned), `R` = reflected velocity. Each bullet tracks `bounces` count; exceeding `maxBounces` (default: 5) destroys the bullet.

### Sweep-Based Bullet Collision
Bullets use continuous collision detection via **swept circle vs. AABB** (Minkowski sum expansion). This prevents tunneling at high speeds.

### Tank Collision
- **Tank vs. walls**: Discrete circle-AABB nearest-point check with overlap pushback (runs after each movement)
- **Tank vs. tank**: O(n²) circle-circle overlap pushback (runs once per tick after all tanks move)

### Maze Generation
Iterative depth-first search (Recursive Backtracker) on a grid where only odd-numbered cells are rooms; even cells are potential passages. The seeded RNG ensures all players generate identical mazes from the same seed, transmitted in the `GAME_START` packet.

---

## Controls

### Local Multiplayer (Default Bindings)
| Player | Move | Rotate | Fire |
|--------|------|--------|------|
| P1 | W / S | A / D | Space |
| P2 | ↑ / ↓ | ← / → | Enter |
| P3 | T / G | F / H | R |
| P4 | I / K | J / L | U |

All bindings are fully customizable in the Local Multiplayer setup screen.

### Online Multiplayer
W/S move · A/D rotate · Space fire (single player per browser)

Press **ESC** during a game to exit (confirmation required).

---

## Configuration

Edit the `C` constants object at the top of the script to tune:
- `TICK_RATE` — simulation Hz (default: 60)
- `SNAP_INTERVAL` — ticks between snapshots (default: 3 = 20 Hz)
- `B_BOUNCES` — bullet bounce limit (default: 5)
- `B_SPEED` — bullet velocity px/s (default: 300)
- `T_SPEED` — tank movement speed (default: 110 px/s)
- `T_FIRE_CD` — fire cooldown seconds (default: 0.45)
- `MAX_PLAYERS` — player cap (default: 8)
- `TIMEOUT` — disconnect timeout ms (default: 7000)
- `COLS`/`ROWS` — maze dimensions (must be odd, default: 19×13)
- `CELL` — maze cell size in pixels (default: 52)

---

## Limitations & Known Constraints

1. **PeerJS cloud dependency**: The WebRTC signaling requires `0.peerjs.com` to be reachable. If blocked by a corporate firewall, the connection will fail. Solution: run the optional local signaling server (not included but PeerJS Server npm package can replace this).

2. **NAT traversal**: Uses Google STUN servers. Symmetric NAT (rare but exists in some enterprise networks) can prevent P2P connection. TURN server support would require a paid relay service.

3. **Host is the bottleneck**: If the host has high latency or packet loss, all clients suffer. This is inherent to host-authoritative design without dedicated servers.

4. **Single online player per browser instance**: Online mode assumes one human per browser tab. For same-machine online testing, open multiple browser tabs with different names.

5. **Join code collisions**: 6-char alphanumeric codes have 34^6 ≈ 1.5 billion combinations. The code tries an alternate if the first PeerJS peer ID is taken.
