# ECONOMY.md — Saltmarket

The authoritative spec for goods, prices, fees, sinks, faucets, and the formulas that make the economy self-balancing. **All numbers here are starting points, not truth.** They must be tuned with the headless sim (Section 11) before UI polish. Every value lives in `packages/shared/src/config/economy.ts` and nowhere else.

---

## 1. Units and time

| Concept | Value |
| --- | --- |
| Currency | **marks**, stored as integer **pips** (1 mark = 100 pips). No floats for balances, ever |
| Tick | 1 game hour. The sim advances in whole ticks |
| Day | 24 ticks |
| Default real speed | 1 tick = 2.5 s real time, so 1 day = 60 s (configurable; headless runs ignore real time) |
| RNG | Seeded PRNG (e.g., mulberry32), passed explicitly; never `Math.random()` |

## 2. Goods

Five goods. Token ids (used in the NEP-245 contract) in the second column.

| Good | Token id | Type | Anchor price (marks) | Notes |
| --- | --- | --- | --- | --- |
| Fish | `fish` | Raw | 10 | Spoils (4%/day of held stock) |
| Salt | `salt` | Raw | 6 | Does not spoil |
| Timber | `timber` | Raw | 8 | Does not spoil |
| Barrel | `barrel` | Intermediate | 30 | Consumed when curing; made by Cooper or players |
| Cured fish | `cured_fish` | Valuable | 45 | Spoils slowly (0.5%/day); what the Captain buys |
| *(Currency)* Marks | `marks` | Currency | 1 | Also a token id on the ledger |
| *(Venue right)* Hall license | `hall_license` | Unique | — | Quantity 1 per Hall; transferable in principle |

**Anchor price** = rough long-run reference used for NPC rules and sanity checks. It is **not** a fixed price. Actual prices are set by orders.

## 3. Recipes (crafting)

| Recipe | Inputs | Output | Duration | Notes |
| --- | --- | --- | --- | --- |
| Craft barrel | 3 timber | 1 barrel | 3 ticks | Input cost about 24 marks; sells near 30 |
| Cure fish | 6 fish + 3 salt + 1 barrel | 4 cured fish | 6 ticks | Input cost about 60 + 18 + 30 = 108 marks, so about 27 per cured fish; sells near 45 |

Baseline margin on curing: about 18 marks per cured fish before taxes, before labor/time, enough to be attractive but narrow enough that input price spikes can wipe it out. That is intentional: **price spikes in inputs must be able to make curing unprofitable**, which then feeds back into supply.

A stall can run one craft at a time (basic). Upgrades add a second slot.

## 4. Starting state

**Player kit:** 300 marks, 5 fish, 2 salt, 0 barrels, basic stall (rent grace 3 days).

**NPC roster and seed values (starting points):**

| NPC | Count | Starting cash (marks) | Starting stock | Base output / behavior |
| --- | --- | --- | --- | --- |
| Fisherman | 3 | 200 | 10 fish | Base catch 8 fish/day |
| Salt-panner | 2 | 150 | 12 salt | Base output 6 salt/day |
| Woodcutter | 2 | 150 | 15 timber | Base output 10 timber/day |
| Cooper | 1 | 300 | 6 timber, 2 barrels | Buys timber, crafts barrels |
| Smokehouse | 1 | 600 | 10 fish, 6 salt, 2 barrels | Buys inputs, cures, sells cured fish |
| Ship Captain | 1 | (treasury is a minter, see §5) | — | Buys cured fish; see §7 |

Starting marks come from a one-time **genesis mint** to NPC and player accounts; this is logged as the initial supply.

## 5. Faucets and sinks (the heart of sustainability)

**Rule: every creation or destruction of marks or goods is an explicit logged event.**

### Faucets (create)
| Faucet | What it creates | Rate / size | Why it exists |
| --- | --- | --- | --- |
| NPC producers | Raw goods (fish, salt, timber) | Base rates in §4, scaled by price response (§8) | Supplies the economy with raw materials |
| **Ship Captain purchases** | **Marks** (the only recurring source of new marks) | Up to `captain.volumePerVisit` cured fish per visit at the clearing price | Ties the money supply to real demand for finished goods |
| **Player gathering nodes** | Raw goods: fish (Pier), salt (Pans), timber (Grove) | Shared nodes with cap and daily regeneration: fish 40 / +20 per day, salt 24 / +10, timber 30 / +15. A gather needs a 4-tick cooldown and a per-player cap of 12 per day. Yield: 2 units if node ≥ 50% full, 1 unit if 15–50%, 0 below 15% | Satisfies "players gather materials"; a commons that can be flooded but is bounded by regeneration |
| Genesis mint | Starting marks | One-time | Initial supply |

### Sinks (destroy)
| Sink | Destroys | Starting rate | Notes |
| --- | --- | --- | --- |
| Stall rent | Marks | 15 marks/day per stall (after 3-day grace) | Burn |
| NPC living costs | Marks | 15 marks/day per NPC | Burn; makes NPC bankruptcy possible |
| NPC equipment upkeep | Marks | Fisherman 20, salt-panner 12, woodcutter 12, cooper 10, smokehouse 15 marks/day | Burn |
| Hall license | Marks | 400 marks, one time | Burn |
| Fish spoilage | Goods | 4%/day of held fish | Seeded stochastic rounding |
| Cured-fish spoilage | Goods | 0.5%/day | Rewards fast selling |
| Craft inputs | Goods | Consumed by recipes | Normal conversion |
| Demurrage (hoarding dock fee) | Marks | 0.5%/day of cash above 1,000 marks per account | Burn; prevents unbounded accumulation |

### Recycling (neither created nor destroyed)
| Flow | Detail |
| --- | --- |
| **Harbor tax** | 2% of each trade's value, paid by the seller to the **Town Fund** |
| **Town Fund** | Pays the rescue seed for merchants nobody invests in (§8.6) and storm relief; excess above a cap (5,000 marks) is burned weekly |
| Hall fee | 0–5% of trade value, paid by the seller to the Hall owner |

### Balance target
Over any rolling 5-day window after warm-up, **marks burned ÷ marks minted should be between 0.85 and 1.10**. Levers when out of range: Captain volume/ceiling, living costs, upkeep, rent, demurrage rate, harbor tax cap. Tune via `pnpm sim:headless`.

## 6. Markets and venues

### 6.1 Harbor Market (default, run by the town)
- **Mechanism:** posted prices. Each seller lists goods at an ask. Buyers pick listings.
- **Friction:** each buyer can complete **one** purchase per tick at stalls; no standing bids.
- **Cost:** harbor tax (2%) only; no hall fee.

### 6.2 Exchange Hall (player-owned)
- **Mechanism:** continuous double auction per good (standing bids and asks, matched by price then time priority).
- **Advantages:** unlimited trades per tick, standing bids, depth and trade-history visible.
- **Cost:** harbor tax (2%) + hall fee (0–5%, owner-set, charged to seller).
- **Fee changes** take effect from the next tick; open orders are unaffected until filled (the fee at match time applies).

### 6.3 Venue choice (NPCs and players)
Each actor computes an **effective cost** per venue:
`effective_price(venue, side) = price × (1 ± (tax + fee)) + friction_penalty(venue)`
where `friction_penalty` for the Harbor Market is a small constant (default 1.5% of price) representing the one-purchase-per-tick limit and search cost. NPCs choose the venue with the best effective price (ties to the Harbor Market). This is what makes Hall competition real.

### 6.4 Matching rules (Halls)
- Price-time priority. Trades execute at the **resting order's** price.
- Minimum order size 1; integer quantities.
- A trade fails atomically (no partial state) if either side lacks funds/goods at match time.
- **Self-trade prevention:** an order cannot match an order from the same account, or from accounts flagged as the same owner (e.g., a player and their own stall).

## 7. The Ship Captain (demand engine)

| Parameter | Start value |
| --- | --- |
| Visit interval | every 48 ticks (2 days) |
| Dock duration | 12 ticks |
| Volume per visit | up to 16 cured fish |
| Price ceiling at first visit | 60 marks |
| Ceiling adjustment | After a visit: if fully filled within 6 ticks, ceiling −5%; if unfilled ≥ 25%, ceiling +10% |
| Ceiling bounds | min 35, max 90 marks |

**Buying rule:** while docked, the Captain buys the cheapest available cured-fish asks (across all venues, after fees) at or below the ceiling until volume is met or the ship leaves. Payment is **minted marks** (faucet). Unmet demand is simply lost (the ship sails away).

Interpretation: the ceiling acts as a price signal. When cured fish is scarce, the ceiling climbs, curing becomes more profitable, and more NPCs/players enter; when supply is abundant, the ceiling drops.

## 8. NPC pricing and production formulas

All NPC behavior is deterministic given the state and seed. Formulas are intentionally simple so judges (and tests) can reason about them.

### 8.1 Ask price for a seller NPC
```
target_stock    = archetype.targetStock            // e.g., Fisherman: 12 fish
stock_pressure  = (target_stock - inventory) / target_stock   // >0 means low stock
cost_basis      = archetype.costBasis(good)        // anchor-based unit cost incl. upkeep share
demand_pressure = clamp(units_sold_last_24t / expected_daily_sales - 1, -0.5, 1.0)  // >0 selling fast, raise
raw_ask         = cost_basis * (1 + margin) * (1 + k_stock * stock_pressure) * (1 + k_demand * demand_pressure)
ask             = clamp(raw_ask, 0.5 * anchor, 2.5 * anchor)
```
Defaults: `margin` per NPC personality (0.08–0.20), `k_stock = 0.4`, `k_demand = 0.25`. This is the "sets prices from its stock and recent demand" requirement. The code for this lives only in `packages/sim/src/pricing/merchant-rules.ts` (see §8.6).

### 8.2 Bid price for an input buyer (Cooper, Smokehouse)
```
expected_output_value = recent_median_price(output_good)           // last 2 days
other_costs           = sum(other input costs) + upkeep_share
max_bid               = expected_output_value / units_out_per_input - other_costs_per_input
bid                   = min(max_bid * (1 - margin), spendable_cash_per_tick_limit)
```
If `max_bid ≤ 0` the NPC **pauses buying** and emits a reasoning line ("Curing isn't profitable at current salt prices; pausing purchases").

### 8.3 Production response (price elasticity of supply)
```
price_ratio   = recent_median_price(good) / anchor(good)
rate_factor   = clamp(price_ratio ^ elasticity, 0.3, 1.8)       // elasticity default 0.8
daily_output  = base_output * rate_factor * weather_factor * (1 - fatigue)
```
Producers slow down when prices fall (below cost they may stop) and speed up when prices rise. `weather_factor` is 1.0 normally and 0.15 during a Storm day.

### 8.4 Inventory and cash limits
- NPC spends at most `maxSpendPerDay` (see NPC_AGENTS §6) and never below `cashFloor` (living costs for 3 days).
- NPC sells down excess inventory when stock exceeds 2 × target (discount of up to 15%), because spoilage punishes hoarding.

### 8.5 Market glut response ("flood the market and it gets cheap")
```
glut     = clamp(listed_supply(good) / avg_listed_supply_2d(good) - 1, 0, 1.5)
bid      = max_bid * (1 - k_glut * glut)            // k_glut default 0.3
```
Merchants that buy a good lower their bids as supply floods in, and stop buying when their stock exceeds 2× target. Sellers' asks fall through order-book competition, and producers cut output via elasticity (§8.3). **Test (scenario):** scripted bots dump 200 timber in one day → timber median falls ≥ 25% within 1 day, woodcutters cut output, and prices recover over about 3–5 days.

### 8.6 Merchant closure, investment, and the one-file rules

**Closure.** A merchant whose cash is below one day's living cost for 3 consecutive days with no sellable inventory goes **CLOSED**: it cancels orders, stops buying/selling/producing, and its card shows "Closed: needs investment". Event `MERCHANT_CLOSED` is emitted with the reasons.

**Investment.** Any player may invest between 150 and 600 marks (config) in a closed merchant:
- The principal moves to the merchant's wallet (a recycled transfer, not minted) and the merchant reopens (`MERCHANT_REOPENED`).
- The investor receives **25% of the merchant's positive daily net profit** (paid from the merchant's cash as a transfer) until total payouts reach **125% of principal**, then the stake ends.
- One investor per merchant at a time. Investing in a shop with no hope of profit loses money: that risk is intentional.

**Rescue fallback.** If nobody invests within **5 game days**, the Town Fund seeds the merchant with up to 150 marks (no profit share) so the town cannot die; flagged `MERCHANT_RESCUED` in the feed.

**One readable rules file.** All merchant pricing and quantity decisions live in `packages/sim/src/pricing/merchant-rules.ts`: a declarative table per archetype (inputs, formulas from §8.1–8.5, clamps, personality hooks) plus an `explain()` function that produces the reasoning text used by NPC cards and tooltips. Rules elsewhere in the code may *call* it but must not duplicate or override it. A test should fail if price-setting logic appears outside that file.

## 9. Anti-exploit rules

| Exploit | Mitigation |
| --- | --- |
| Wash trading to farm Hall fees | Self-trade prevention; fees are paid by the seller to the owner, so wash trading costs the harbor tax to the Town Fund; net negative |
| Multi-account wash trades | Fees and tax make cycles lossy; additionally, rate limit trades between the same account pair (e.g., max 10/day) |
| Arbitrage between venues | Allowed and encouraged; friction + fees keep it modest |
| Cornering a good | Spoilage (fish) and demurrage limit hoarding; NPC price response and supply elasticity limit it |
| Gathering abuse (bots, many accounts) | Node regeneration bounds total supply regardless of account count; per-player daily cap; cooldown; yield falls as nodes deplete |
| Investment scams (invest, drain the shop) | Investors can't withdraw principal; payouts come only from positive daily net profit; one investor per merchant; investment cannot come from the merchant's own accounts |
| Hall fee 5% + price manipulation | Competitors undercut; NPC venue choice favors cheaper venues |
| Infinite money from the Captain | Volume cap per visit; ceiling reacts; sinks recover |

## 10. Invariants (enforced by tests)

1. **Mark conservation:** `sum(all account marks incl. Town Fund) = genesis + minted − burned`, always, at every tick.
2. **Good conservation:** per good, `sum(inventories) = produced + gathered − consumed − spoiled`, always.
3. **No negative balances** or inventories.
4. **Determinism:** same seed + same inputs → byte-identical event log hash after N ticks.
5. **Fees add up:** for each trade, `buyer_paid = seller_received + tax + fee`.
6. **Hall fee bound:** `0 ≤ fee ≤ 5%`.
7. **Captain never buys above ceiling.**

## 11. Tuning procedure (do this before UI polish)

1. Implement the sim with the config above.
2. Run `pnpm sim:headless --seed S --days 30` over at least 10 seeds with **no players** (NPCs only), then with 5 scripted "bot players" using naive strategies.
3. Check:
   - No account reaches zero permanently unless it is an NPC that then respawns.
   - No price stays pinned at the 0.5× or 2.5× anchor clamp for more than 3 days outside a Storm.
   - Burn/mint ratio within 0.85–1.10 after day 5.
   - Total marks do not grow or shrink more than ±30% over days 5–30.
   - Captain ceiling oscillates within bounds rather than pinning at an edge.
4. If out of range, adjust **one lever at a time** and record the change and rationale in a `docs/TUNING_LOG.md` (create it at the first tuning run).
5. Run the **Storm scenario** (see `TESTING_AND_DEMO.md`) and confirm the recovery pattern.

## 12. Metrics the sim must expose (also feeds the UI and the judge panel)

Per tick and per day: median/last price per good, volume per good per venue, inventories (aggregate and per NPC), marks supply (total, minted, burned), Town Fund balance, Captain ceiling and fill rate, NPC cash/bankruptcies, Hall fee income per Hall, and a short rolling event log.

## 13. Config shape (reference)

```ts
export const ECONOMY = {
  time: { ticksPerDay: 24, realMsPerTick: 2500 },
  currency: { pipsPerMark: 100 },
  goods: {
    fish:       { anchor: 1000, spoilPerDay: 0.04 },
    salt:       { anchor: 600,  spoilPerDay: 0 },
    timber:     { anchor: 800,  spoilPerDay: 0 },
    barrel:     { anchor: 3000, spoilPerDay: 0 },
    cured_fish: { anchor: 4500, spoilPerDay: 0.005 },
  },
  recipes: {
    craft_barrel: { in: { timber: 3 }, out: { barrel: 1 }, ticks: 3 },
    cure_fish:    { in: { fish: 6, salt: 3, barrel: 1 }, out: { cured_fish: 4 }, ticks: 6 },
  },
  fees: { harborTaxBps: 200, hallFeeMaxBps: 500, harborFrictionBps: 150 },
  sinks: { stallRentPerDay: 1500, hallLicense: 40000, demurrage: { thresholdMarks: 1000, bpsPerDay: 50 } },
  captain: { everyTicks: 48, dockTicks: 12, volume: 16, startCeiling: 6000, minCeiling: 3500, maxCeiling: 9000 },
  townFund: { weeklyBurnCap: 500000 },
  elasticity: 0.8,
  pricing: { kStock: 0.4, kDemand: 0.25, kGlut: 0.3 },
  gather: { cooldownTicks: 4, perPlayerPerDay: 12,
    nodes: { fish: { cap: 40, regenPerDay: 20 }, salt: { cap: 24, regenPerDay: 10 }, timber: { cap: 30, regenPerDay: 15 } } },
  merchants: { closeAfterDaysBroke: 3, investMin: 15000, investMax: 60000, profitShareBps: 2500, payoutCapBps: 12500, rescueAfterDays: 5, rescueSeed: 15000 },
  timelapse: { ticks: 168, targetWallSeconds: 60 },
} as const;   // all money values in pips
```
