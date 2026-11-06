# GAME_DESIGN.md — Saltmarket

What the player does, sees, and feels. Numbers and formulas live in `ECONOMY.md`; agent logic lives in `NPC_AGENTS.md`. If a number here differs from `ECONOMY.md`, `ECONOMY.md` wins.

---

## 1. Player fantasy and tone

You are a newcomer to a small harbor town with a tiny stall and a little coin. Over a few sessions you become a trader, a curer, or the owner of the town's busiest exchange. The tone is **cozy, readable, and a little wry**. The town feels inhabited because its merchants talk about what they are doing and why.

**Design pillars:**
1. **Legible economy.** Every price move should be explainable in one sentence on screen.
2. **Short loop, deep consequences.** The loop takes a minute to learn; the interesting part is how the town reacts.
3. **Agents with personalities.** NPCs should feel like characters with margins, fears, and habits.
4. **Invisible chain.** Nothing in the player's flow mentions crypto.

## 2. The core loop

```
   ┌─────────► BUY raw goods (fish, salt, timber)
   │                     │
   │                     ▼
REINVEST            CRAFT / CURE (cured fish needs fish + salt + a barrel)
(bigger stall,            │
 more stock,              ▼
 a Hall license)     SELL to the ship captain, NPCs, or other players
   ▲                     │
   └──────── PROFIT ◄────┘
```

One pass of the loop takes about 1–3 game days (60–180 real seconds at default speed). A session of 10 minutes covers roughly 10 game days and a couple of ship visits. The 30-second, 5-minute, and per-session breakdown is in `GAME_LOOP.md`.

**Player decisions that matter:**
- What to buy now vs. wait for a better price.
- Whether to cure yourself or sell raw goods to the Smokehouse NPC.
- When to hold stock for the next captain visit.
- Which venue to trade in (free Harbor Market with friction vs. a Hall with a fee but faster, deeper matching).
- Whether to invest in a Hall license.

## 3. The first minute (onboarding)

Target: a new player completes a first profitable trade within 60 seconds.

1. **Sign in** with a social login (single click). No wallet language.
2. **Welcome card (3 lines max):** "You have a stall, 300 marks, and a few goods. Buy low, craft, sell high. The ship arrives in about 2 days."
3. **Guided first trade** (dismissible hints, not a locked tutorial):
   - Highlight a fisherman's fish offer: "Buy fish here."
   - Highlight the Cure button: "Cure fish with salt and a barrel to make something ships want."
   - Highlight the ship timer: "Sell cured fish to the captain when the ship docks."
4. Starting kit: 300 marks, 5 fish, 2 salt, 0 barrels, a basic stall (3-day rent grace).

## 4. Actions available to a player

| Action | Where | Notes |
| --- | --- | --- |
| Buy | Harbor Market stall listings or Hall order book | Pays seller, harbor tax, and Hall fee (if a Hall) |
| Sell / list | Same | Post an ask price, or accept an existing bid |
| Place bid | Hall order book | Standing buy order for a good at a price |
| Cure | Own stall | Recipe: 6 fish + 3 salt + 1 barrel → 4 cured fish (takes 6 ticks; see ECONOMY §3) |
| Craft barrel | Own stall | 3 timber → 1 barrel (3 ticks); competes with the Cooper NPC |
| Buy Exchange Hall license | Town screen | One-time cost; then set fee 0–5% |
| Set Hall fee | Hall screen | Owner-only; changes apply from the next tick |
| Upgrade stall | Stall screen | More inventory slots and a small crafting-speed bonus |
| **Gather** | Pier (fish), Pans (salt), Grove (timber) | Shared nodes; 4-tick cooldown; yield falls as a node is depleted; regenerates daily (ECONOMY §5) |
| **Invest in a closed shop** | Closed merchant's card | Put marks into a broke merchant to reopen it; earn a profit share (ECONOMY §8.6) |
| Watch a week | Top bar | Read-only time-lapse of 7 game days in about 60 s (GAME_LOOP §6) |

Raw goods enter the economy through **NPC producers** and through **player gathering** at three shared nodes (Pier, Pans, Grove). Nodes have a finite stock that regenerates daily, so gathering is a commons: crowded nodes yield less, and a market flooded with one good sees its price fall. Gathering is capped per player per day so it supplements trading rather than replacing it.

## 5. Roles

### Stall Keeper (default)
Buys, crafts, and sells from their own stall. Pays daily rent. Competes with NPC Smokehouse on cured fish.

### Hall Owner
Buys an Exchange Hall license. A Hall is a **venue** with:
- an order book per good (continuous matching),
- an owner-set fee (0–5%, charged to the seller),
- a price board and recent-trades feed visible to anyone in that Hall,
- throughput advantages over the Harbor Market (see ECONOMY §6).

Why would anyone use a Hall and pay a fee? Because the Harbor Market has friction (see below). Why would anyone own one? Because trade volume × fee is the best passive income in town, *if* the owner prices competitively and attracts liquidity. Rival halls undercut each other; NPCs choose venues by effective cost, so the market genuinely shifts.

### Investor
When a merchant goes broke it **closes its shop** and shows "Closed: needs investment". Any player can invest marks to reopen it and receives a share of the merchant's profits until repaid with a return (ECONOMY §8.6). Investors need to read the market: a shop that failed because of a storm is a good bet; one that failed because its inputs are structurally too expensive is not. Merchants' closure and reopening appear in the event feed.

### Harbor Market (the default venue, run by the town)
Free to list, but: posted prices only (no order book), each actor can complete only one purchase per tick at stalls, and the 2% harbor tax applies. It is always available so the game never deadlocks without a Hall.

## 6. The town and its NPCs

| NPC archetype | Count | What they do | What makes them interesting |
| --- | --- | --- | --- |
| Fisherman | 3 | Catch fish, sell fish | Fish less when prices drop; weather-sensitive |
| Salt-panner | 2 | Produce salt, sell salt | Raise prices when stock is low |
| Woodcutter | 2 | Produce timber, sell timber | Steady, low-margin supplier |
| Cooper | 1 | Buy timber, craft and sell barrels | Reacts to barrel demand from curers |
| Smokehouse | 1 | Buy fish + salt + barrels, cure, sell cured fish | Main competitor for player curers |
| Ship Captain | 1 (visits) | Buys cured fish in bulk every ~2 days | The demand engine; raises/lowers his price ceiling by results |

Each NPC shows a **reasoning line** on its card (e.g., *"Fish stock is low and the storm is coming. Raising my price 12%."*). Full behaviors in `NPC_AGENTS.md`.

## 7. Events

| Event | Trigger | Effect | Purpose |
| --- | --- | --- | --- |
| **Storm** | Judge/dev button, or random (low probability) | Fish catch drops about 85% for 1 day, plus 25% for the next day | The headline "economy absorbs a shock" demo |
| Fair Winds | Random | Captain visit brings extra demand (+50% volume) | Positive shock, shows boom behavior |
| Salt Glut | Random | Salt output +60% for 1 day | Shows price collapse and NPC response |

Events are logged to a feed with their cause and effect. The feed is a core part of legibility.

## 8. Progression and goals

There is no "win". Motivation comes from:
- **Net worth** on a town leaderboard (marks + goods at last-trade prices).
- **Town prosperity** meter (aggregate trade volume), shown as a communal goal.
- **Hall fee income** leaderboard.
- **Milestones**: first cure, first captain sale, first Hall license, survived first Storm.

Do not add persistent grind mechanics. Keep sessions meaningful at 5–15 minutes.

## 9. UI screens (web, responsive)

1. **Town view (home):** top bar with marks, day/time, and ship countdown. Left: your stall (inventory, craft buttons). Center: market panel with goods, current best bid/ask, and ▲/▼ trends. Right: NPC cards with reasoning lines. Bottom: event feed.
2. **Market panel:** per-good price board with sparkline (last 5 days), depth, and a Buy/Sell dialog. Venue selector (Harbor Market / Hall A / Hall B) with each venue's effective fee shown.
3. **Stall screen:** inventory, craft queue with progress bars, rent due, upgrade button.
4. **Hall screen (owner):** set fee, see volume and fee income chart, see competing halls' fees.
5. **NPC detail drawer:** wallet balance (as "coin purse"), inventory, last 5 decisions with reasons.
6. **Leaderboard:** net worth, Hall income, town prosperity.
7. **Judge/Dev panel (opt-in):** Storm and event buttons, speed control, seed display, the **Ledger drawer** (settlements, tx links, queue status).

8. **Time-lapse panel ("Watch a week"):** price lines for all goods, merchants' cash and open/closed state, Captain ceiling, marks supply, and a scrolling feed of notable reasoning lines. Optional toggles for scripted shocks (storm, flood, closure). Runs on a sandbox clone (ARCHITECTURE §3.5).

**Player-facing vocabulary:** "marks", "stall", "purse", "buy", "sell", "ship". **Forbidden in player UI:** wallet, token, gas, mint, tx, address, NEAR, blockchain. (Allowed only in Judge/Dev panel and README.)

## 10. Art and audio direction

- Minimal 2D with a limited warm palette; flat shapes and good typography; no dependency on external art assets (use inline SVG/emoji-style icons for goods: 🐟 🧂 🪵 🛢️ 🍣 is acceptable placeholder language, replace with simple SVG if time allows).
- Day/night tint tied to tick for a sense of time.
- Optional soft UI sounds; muted by default.

## 11. Feel and feedback

- **Every merchant price and NPC card has a reasoning tooltip** generated from the rules file (`merchant-rules.ts`). Price changes animate subtly; the reason appears in the tooltip ("Fish ▲ 12%: storm reduced catch").
- Every trade shows a one-line confirmation ("Sold 4 cured fish to Captain Maren for 180 marks").
- Failures are friendly and specific ("Not enough salt: you need 3, you have 2").
- Never show a spinner for chain settlement; the UI is optimistic and server-authoritative.

## 12. Edge cases to design for

- **Player has zero marks and no goods:** gathering is free and always available (subject to node stock and the daily cap), so no one is ever locked out. If nodes are depleted, the player waits for regeneration or works for a merchant bid.
- **NPC goes bankrupt:** it **closes its shop until a player invests** (NPC_AGENTS §8). If nobody invests within 5 game days, the Town Fund rescues it so the town does not die; the feed marks it as rescued. Show closures, investments, and rescues in the feed.
- **Hall owner sets fee to 0%:** allowed. Hall still earns reputation as the cheap venue; NPCs shift volume to it. Harbor tax still applies.
- **Wash trading:** self-trades earn no fee income and are rejected (ECONOMY §9).
- **Player inactive:** stall keeps paying rent from their purse; if purse hits zero for 5 days the stall is "shuttered" (inventory preserved, no craft or listing) until they return and pay arrears.
