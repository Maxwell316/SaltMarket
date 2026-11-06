# ARCHITECTURE.md — Saltmarket

System design for the game server, simulation, client, and the ledger seam. Chain specifics are in `NEAR_INTEGRATION.md`.

---

## 1. Overview

```
 ┌────────────┐   WebSocket / REST    ┌───────────────────────────────────────────┐
 │  Web client│◄─────────────────────►│                Server (Node)              │
 │ (Vite/React)│                      │  ┌──────────┐   ┌──────────────────────┐  │
 └────────────┘                       │  │ Tick loop│──►│  packages/sim (pure) │  │
                                      │  └────┬─────┘   └──────────┬───────────┘  │
                                      │       │ events             │ state        │
                                      │  ┌────▼─────┐        ┌─────▼──────┐       │
                                      │  │Event log │        │ Snapshots  │ SQLite│
                                      │  └────┬─────┘        └────────────┘       │
                                      │       │ settlement intents                │
                                      │  ┌────▼─────────────────────────────┐     │
                                      │  │ Settlement queue (write-behind)  │     │
                                      │  └────┬─────────────────────────────┘     │
                                      │  ┌────▼─────────┐     ┌────────────────┐  │
                                      │  │ LedgerAdapter│────►│ Relayer (gasless)│ │
                                      │  │ Memory | NEAR│     └───────┬────────┘  │
                                      │  └──────────────┘             │           │
                                      └───────────────────────────────┼───────────┘
                                                                      ▼
                                                              NEAR testnet
                                                    (NEP-245 goods contract, sub-accounts)
```

**Key idea:** the sim is the game. The chain is a ledger that mirrors ownership and settles value. The UI talks to the server only; it never touches the chain directly.

## 2. Packages and responsibilities

| Package / app | Responsibility | May import |
| --- | --- | --- |
| `packages/shared` | Types, zod schemas, constants, the single economy config | nothing internal |
| `packages/sim` | Deterministic engine: markets, production, spoilage, events, NPCs, metrics | `shared` |
| `packages/chain` | `LedgerAdapter` interface, `MemoryLedger`, `NearLedger`, settlement batching, account helpers | `shared` |
| `apps/server` | Tick loop, session/auth, WS+REST API, persistence, settlement queue, relayer | `shared`, `sim`, `chain` |
| `apps/web` | UI | `shared` (types only), never `sim` internals or `chain` |
| `contracts/goods` | Rust NEP-245 contract | — |

Rule: **`sim` has no I/O and no dependency on `chain`.** The server translates sim events into ledger intents.

## 3. The simulation (packages/sim)

### 3.1 Core signature
```ts
step(state: WorldState, inputs: PlayerInput[], rng: Rng): { state: WorldState; events: SimEvent[] }
```
One call = one tick. Pure and deterministic. `inputs` are validated, ordered player commands collected since the last tick.

### 3.2 Order of operations within a tick (fixed, documented, tested)
1. Apply scheduled events (Storm start/end, ship arrival/departure).
2. Apply player inputs in deterministic order (by arrival sequence number).
3. Run NPC `observe → decide → guard → act` (staggered per NPC).
4. Production and crafting progress (completions emit goods).
5. Market matching: Hall books (price-time), then Harbor Market purchases (one per actor).
6. Captain logic (if docked).
7. Daily boundary (tick % 24 == 0): rent, living costs, upkeep, demurrage, spoilage, Town Fund housekeeping, bankruptcy checks, NPC strategy review, Captain ceiling update, rollover metrics.
8. Emit metrics snapshot and the event list.

### 3.3 State shape (sketch)
```ts
type WorldState = {
  tick: number; seed: number; rngState: number;
  accounts: Record<AccountId, { kind: 'player'|'npc'|'townfund'|'captain'; marks: Pips; inv: Record<Good, number> }>;
  stalls: Record<StallId, Stall>;
  venues: Record<VenueId, Venue>;           // harbor + halls; halls hold order books
  orders: Record<OrderId, Order>;
  npcs: Record<NpcId, NpcState>;
  ship: ShipState;
  events: ActiveEvent[];                    // Storm, etc.
  supply: { genesis: Pips; minted: Pips; burned: Pips };
  metrics: RollingMetrics;
};
```
Use plain JSON-serializable data (no class instances, Maps, or Dates) so snapshots and replays are trivial.

### 3.4 Events
Every state change that matters emits a typed `SimEvent` (`TRADE`, `MINT`, `BURN`, `PRODUCE`, `CRAFT_DONE`, `SPOIL`, `RENT`, `FEE_PAID`, `STORM_START`, `NPC_DECISION`, `BANKRUPTCY`, `HALL_LICENSED`, ...). The event log is the audit trail, the settlement source, and the UI feed source.

### 3.5 Time-lapse sandbox ("Watch a week", the fast-forward mode)

Requirement: *simulate a week of trading in a minute so judges can watch the economy move.*

- **Trigger:** `POST /api/timelapse` or WS `{ t:'timelapse', action:'start', days:7, shocks? }`.
- **How:** the server deep-clones the current `WorldState` (it is plain JSON, so this is trivial) including `seed` and `rngState`, creates a **sandbox** with its own `MemoryLedger`, and steps `step()` for `ticks = days * 24` (default 168).
- **Pacing:** compute as fast as needed, but emit frames paced to about `TIMELAPSE_WALL_SECONDS` (default 60; configurable 30–120) so the UI animation is watchable. Send a **frame every 6 ticks** to the requesting client only: prices and volume per good, merchants' cash and open/closed state, Captain ceiling, marks supply and burn/mint ratio, notable reasoning lines.
- **Isolation:** the sandbox never writes to the live world, the event log, or the chain. The live world keeps ticking independently. Player commands are ignored inside the sandbox.
- **Scripted shocks (optional toggles):** storm on day N, timber flood on day N, close a merchant on day N and have an investor bot reopen it on day M. Use the normal event machinery so the behavior is the real behavior.
- **Merchant closures inside the sandbox** resolve via investor bots or the Town Fund rescue (NPC_AGENTS §8).
- **Headless equivalent:** `pnpm sim:week --seed 42 --storm-day 3` prints the same story as a table and exits non-zero if invariants fail.
- **Budget:** keep sandbox CPU bounded (max 1 concurrent time-lapse per client, 3 per server) so judges cannot degrade the live game.

### 3.6 The merchant rules file

All merchant price and quantity decisions are made in `packages/sim/src/pricing/merchant-rules.ts` and nowhere else (format in `NPC_AGENTS.md` §5.1). The sim's NPC step calls `decideMerchant(archetype, obs, personality)`, which returns orders plus a `Reasoning`. The tooltip API uses `explain()` from the same module. Enforce with a lint or test that scans for price computations outside the file.

## 4. Server (apps/server)

### 4.1 Tick loop
- A scheduler calls `step` at `realMsPerTick`. Supports pause, speed multiplier (1×, 4×, 16×, "headless max") for demos.
- Player commands arrive via WS, are validated and enqueued with a sequence number, and are consumed on the next tick.
- After each tick: persist events, push a **state delta** to clients, and push settlement intents to the queue.

### 4.2 Persistence (SQLite via `better-sqlite3`)
- `events(seq INTEGER PK, tick INTEGER, type TEXT, payload JSON)`: append-only.
- `snapshots(tick INTEGER PK, state JSON)`: every N ticks (default 24) and on graceful shutdown.
- `sessions(id, playerId, createdAt, ...)` and `players(id, displayName, nearAccountId, ...)`.
- `settlements(id, tickFrom, tickTo, status, txHash?, attempts, error?)`: the queue's durable record.
- On boot: load the latest snapshot, then replay later events if needed. A `--fresh --seed N` flag starts a new world.

### 4.3 Auth and sessions
- Sign-in flow is an **adapter** (`AuthProvider`): `SocialLoginAuth` (real) and `GuestAuth` (dev/demo fallback, no external dependency). Details in `NEAR_INTEGRATION.md` §5.
- The server maps a session to a `playerId` and to a ledger account id. The client never handles keys.

### 4.4 API surface

**REST (JSON):**
| Method & path | Purpose |
| --- | --- |
| `POST /api/auth/login` | Begin/complete sign-in; returns session |
| `GET /api/world` | Full current snapshot for initial load |
| `GET /api/market/:good/history?days=5` | Price history for sparklines |
| `GET /api/npcs` | NPC cards with last decisions |
| `GET /api/leaderboard` | Net worth, Hall income, prosperity |
| `GET /api/judge/ledger?limit=50` | Recent settlements with explorer links (judge mode) |
| `POST /api/dev/event` | Trigger Storm/Fair Winds/Salt Glut (**dev/judge flag only**) |
| `POST /api/dev/speed` | Set sim speed (**dev/judge flag only**) |
| `POST /api/timelapse` | Start a sandbox time-lapse ("Watch a week"); frames stream over WS |
| `GET /api/npcs/:id/explain` | Tooltip payload: reason text + key factors, generated by the rules file |
| `GET /healthz` | Liveness, tick number, ledger status |

**WebSocket messages (client → server), all zod-validated:**
```ts
{ t: 'buy',    good, qty, venueId, maxPrice? }
{ t: 'sell',   good, qty, venueId, minPrice? }
{ t: 'bid',    good, qty, price, venueId }            // Halls
{ t: 'cancel', orderId }
{ t: 'craft',  recipe }
{ t: 'upgradeStall' }
{ t: 'buyHall' }
{ t: 'gather', node: 'pier' | 'pans' | 'grove' }
{ t: 'invest', npcId, amount }                         // reopen a closed merchant
{ t: 'timelapse', action: 'start' | 'stop', days?: 7, shocks?: Shock[] }
{ t: 'setHallFee', hallId, bps }
{ t: 'subscribe', topics: ['world','feed','market:fish', ...] }
```
**Server → client:** `{ t:'delta', tick, changes }`, `{ t:'feed', items }`, `{ t:'ack', cmdId, ok, code?, message? }`, `{ t:'tick', tick, day, hour }`.

Friendly, specific error messages (`code` + `message`), e.g. `INSUFFICIENT_GOODS: you need 3 salt, you have 2`.

### 4.5 Dev/judge gating
Dev routes and the Judge panel are enabled when `JUDGE_MODE=1` (server) and the client is opened with `?judge=1`, so the public demo can show the Storm button on purpose without exposing it to all players by accident.

## 5. The ledger seam (packages/chain)

```ts
interface LedgerAdapter {
  ensureAccount(owner: OwnerRef): Promise<LedgerAccountId>;          // player or NPC or fund
  applyGenesis(allocations: Allocation[]): Promise<Receipt>;         // initial mint
  settle(batch: SettlementBatch): Promise<Receipt>;                  // net transfers + mint/burn
  balanceOf(account: LedgerAccountId, tokenId: TokenId): Promise<bigint>;
  status(): LedgerStatus;                                            // 'ok' | 'degraded' | 'offline', queue depth
}
```
- **`MemoryLedger`**: in-process, instant, used in dev, tests, and the offline demo. Produces fake but well-formed receipts so the Judge drawer logic is testable.
- **`NearLedger`**: talks to NEAR testnet via `near-api-js` (verify API in docs). Uses batched calls.
- Selected by `LEDGER=memory|near` env var.

### 5.1 Settlement strategy (write-behind, batched)
1. The server converts sim events (`TRADE`, `MINT`, `BURN`, `CRAFT_DONE`, `PRODUCE`, `HALL_LICENSED`...) into **net balance deltas per (account, token)**.
2. Every `SETTLE_EVERY_TICKS` (default 6, i.e., a quarter-day) it builds one `SettlementBatch` of net deltas.
3. The adapter submits the batch (transfers via `mt_batch_transfer` and authority-only mint/burn; verify names in docs).
4. On success: record `txHash`, mark settled. On failure: retry with exponential backoff; the queue keeps growing but the **game never blocks**.
5. A **reconciliation job** (every day) compares sim balances to ledger balances for a sample of accounts and logs mismatches (should be empty if the batcher is right).

Net settlement (not per-trade) keeps tx count low and sidesteps rate and cost concerns, while onchain state still reflects ownership at each batch boundary. State this honestly in the README.

## 6. Client (apps/web)

- **State:** a single store (Zustand or similar) hydrated from `GET /api/world`, updated by WS deltas. Optimistic UI only for harmless things (button states); trades render from server acks.
- **Components:** `TopBar`, `StallPanel`, `MarketPanel` (+ `PriceSparkline`, `OrderDialog`), `NpcCards`, `EventFeed`, `HallScreen`, `Leaderboard`, `JudgePanel`, `LedgerDrawer`.
- **Time:** the client displays server tick; interpolates the clock for smoothness but never drives game logic.
- **Charts:** minimal custom SVG sparklines; avoid heavy chart libs unless they save real time.
- **Resilience:** auto-reconnect WS with backoff; show a gentle "reconnecting…" banner; resync with `GET /api/world` after reconnect.

## 7. Configuration and environment

`.env.example` lists names only:
```
PORT=3000
LEDGER=memory                 # memory | near
SEED=42
REAL_MS_PER_TICK=2500
JUDGE_MODE=0
SETTLE_EVERY_TICKS=6
NEAR_NETWORK=testnet
NEAR_AUTHORITY_ACCOUNT_ID=    # human supplies; never invent
NEAR_AUTHORITY_PRIVATE_KEY=   # secret; env only
NEAR_RELAYER_ACCOUNT_ID=
NEAR_RELAYER_PRIVATE_KEY=     # secret; env only
GOODS_CONTRACT_ID=
AUTH_PROVIDER=guest           # guest | social
NPC_LLM_ADVISOR=0
```
Secrets never appear in logs, errors, or client bundles.

## 8. Observability

- Structured logs (pino): `tick`, `ledger`, `settlement`, `npc`, `ws` scopes.
- `/healthz` includes tick, queue depth, last settlement time, ledger status.
- A `metrics` WS topic powers the Judge panel (prices, supply, burn/mint ratio, Captain ceiling).

## 9. Performance expectations

Small world: ~10 NPCs + tens of players; a tick should compute in well under 50 ms. If a tick exceeds budget, log a warning with a per-phase timing breakdown. Do not optimize prematurely, but do not allocate in hot loops needlessly.

## 10. Error handling principles

- Expected failures (insufficient funds, out-of-stock) are **typed results**, never exceptions.
- Invariant violations (negative balance, conservation mismatch) **throw in tests/dev** and log loudly + halt the tick in production demo mode (better a visible pause than silent corruption).
- Ledger errors never propagate into the sim; they only affect the settlement record and the status indicator.

## 11. Deployment (for the demo)

- Single Node process serving the API and static web build. Any simple host works (e.g., a small VM or container platform).
- Always keep a **local one-command run** (`pnpm dev`) with `MemoryLedger` as the guaranteed fallback for judges.
- Do not deploy or publish anything on the human's behalf without confirmation (CLAUDE.md §3).
