# GAME_LOOP.md — Saltmarket (design on paper, before any code)

The organizers asked for the loop to be designed on paper first: **what the player does every 30 seconds, every 5 minutes, and every session.** This is that document. Claude Code must present it to the human and get approval before writing code (see `TRACK_BRIEF.md` §4). Numbers come from `ECONOMY.md`; screens from `GAME_DESIGN.md`.

---

## 1. Time conversions (default speed)

| Real time | Ticks | Game time | What happens in the world |
| --- | --- | --- | --- |
| 2.5 s | 1 | 1 hour | One sim step |
| 10 s | 4 | 4 hours | One gather cooldown |
| 30 s | 12 | Half a day | NPCs post/refresh orders; prices drift |
| 60 s | 24 | 1 day | Daily boundary: rent, upkeep, spoilage, NPC strategy review |
| 2 min | 48 | 2 days | A ship visit |
| 5 min | 120 | 5 days | ~2.5 ship visits; Captain ceiling adjusts |
| 15 min | 360 | 15 days | A full session; a Storm usually lands in this window in judge mode |

## 2. The three loops at a glance

| Loop | Player's question | Core actions | Feedback they get |
| --- | --- | --- | --- |
| **Every 30 s (micro)** | "What should I gather or sell right now?" | Gather at a node, sell a few raw goods, glance at prices and a tooltip | Marks tick up, ▲/▼ price arrows, a merchant's reasoning line |
| **Every 5 min (meso)** | "Can I turn this into something worth more, and when do I sell?" | Assemble a curing batch, cure, time the ship, sell, pay rent, reinvest | Ship sale payout, stall upgrade, a closed shop to invest in |
| **Every session (macro)** | "What kind of trader do I become?" | Choose a path: curer, Hall owner, investor; weather a Storm; climb the leaderboard | Net worth rank, Hall fee income, "while you were away" recap |

## 3. Every 30 seconds — the micro loop

**What the player does (2–4 actions):**
1. **Gather** at a node (Pier for fish, Pans for salt, Grove for timber). 4-tick cooldown (10 s) means about 3 gathers per 30 s. Yield shrinks when a node is crowded or depleted, so choosing *where* to gather is a real choice.
2. **Check the price board** (▲/▼ arrows and best offers) for what they hold.
3. **Sell or hold:** sell surplus raw goods to the best-paying merchant now, or keep them for curing.
4. **Hover a merchant or price:** the tooltip explains *why* ("Salt is scarce, raising my price 12%"). This is the hook that makes the economy feel alive.

**What the economy does meanwhile:** NPCs refresh orders, producers output goods, prices drift with stock and demand, nodes regenerate.

**Why it's engaging:** every 30 seconds there is a tiny decision (where to gather, sell vs. hold) with a visible, explainable result. No dead air.

## 4. Every 5 minutes — the meso loop

**What the player does:**
1. **Assemble a batch:** 6 fish + 3 salt + 1 barrel (gathered, bought, or crafted from 3 timber).
2. **Cure** (6 ticks, 15 s) → 4 cured fish.
3. **Time the ship:** the ship docks every 2 minutes for 30 s. Sell cured fish to the Captain, or to the Smokehouse/other players if the ceiling is poor.
4. **Handle the daily costs:** rent comes due; check the purse.
5. **Reinvest:** upgrade the stall (second craft slot), buy barrels from the Cooper vs. crafting, or stock up before a likely shortage.
6. **Scan the town:** a merchant flashing **Closed: needs investment** is an opportunity (see §5).

**Typical payoff:** one cured batch yields about 18 marks of profit per cured fish at baseline margins, around 70 marks per batch before costs of time, rising when inputs are cheap or the Captain's ceiling is high.

**Why it's engaging:** a planning horizon of a few minutes with a rhythm (ship visits) and a risk (input price spikes can erase curing profit).

## 5. Every session (10–20 minutes) — the macro loop

**Arc of a good session:**
1. **Minutes 0–3:** learn gather → sell → first cure. Grow from 300 marks to roughly 450.
2. **Minutes 3–8:** commit to a path:
   - **Curer:** run batches for the Captain and out-compete the Smokehouse.
   - **Investor:** put marks into a closed merchant to reopen it and earn a profit share (25% of its daily net profit until 125% of principal is repaid).
   - **Hall owner:** buy an Exchange Hall license (400 marks), set a fee, and attract volume.
3. **Minutes 8–15:** a **Storm** (or random event) hits. Prices spike, merchants change behavior, the Captain's ceiling climbs. The player adapts (stockpile, switch venue, invest in a shop hit by the storm).
4. **End:** leaderboard standing (net worth, Hall income, town prosperity), milestones unlocked, and a recap.

**Return hook (between sessions):** the town keeps running. On return, a **"While you were away"** card summarizes events (storms, closures/reopenings, Hall income, investor payouts, rent paid). Stall rent keeps accruing, which gives gentle continuity without punishing short absences (3-day grace, then shuttering per `GAME_DESIGN.md` §12).

**Why it's engaging:** different roles create different economies; players' choices (fees, investments, stockpiling) shape prices for everyone.

## 6. Fast-forward: "Watch a week" (time-lapse)

Required by the track: *a fast-forward mode that simulates a week of trading in a minute, so judges can watch the economy move.*

- **Length:** 7 game days = 168 ticks, paced to about **60 seconds** of wall-clock time (about 2.8 ticks/s; configurable 30–120 s).
- **How it runs:** the server clones the current world into a **sandbox** (same state, same seed), steps it, and streams frames to the requesting client only. It does **not** change the live world, the event log, or the chain. Details in `ARCHITECTURE.md` §3.5.
- **What the judge sees:** a time-lapse panel with price lines for all goods, merchants' cash and open/closed status, the Captain's ceiling, total marks supply, and a scrolling feed of notable reasoning lines ("Smokehouse paused: curing margin is 3%").
- **Optional toggles:** "Add a storm on day 3", "Flood timber on day 2", "Close a merchant on day 4 and let a bot invest on day 5". These make the economy's responses visible in a minute.
- **Headless equivalent:** `pnpm sim:week --seed 42 --storm-day 3` prints the same story as a table.
- **Also available to players** as "Watch the town" (read-only), not only in Judge mode.

## 7. What the player never sees

No wallets, addresses, tokens, gas, signing, or confirmations. Gathering, selling, and investing feel like ordinary game buttons. The only place the chain appears is the opt-in Judge/Ledger drawer.

## 8. Approval gate (process)

Before any code:
1. Claude Code presents §1–§6 to the human in a short summary.
2. The human approves or edits (numbers like cooldowns, ship interval, and rent are tunable).
3. Claude Code records `Loop approved: <date>` in the Status block of `CLAUDE.md` and links any edits back into `GAME_DESIGN.md` and `ECONOMY.md`.

## 9. Open design questions to settle with the human

1. Is a 10 s gather cooldown too fast or too slow for the feel they want?
2. Should investors be able to hold stakes in more than one merchant at a time (default: one investor per merchant, any number of merchants per investor)?
3. Should "Watch a week" be visible to all players or only in Judge mode (default: all players, read-only)?
4. Is a 2-minute ship interval right for 15-minute sessions (default yes; yields 7 or so visits per session)?
