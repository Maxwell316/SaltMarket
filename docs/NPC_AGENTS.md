# NPC_AGENTS.md — Saltmarket

How NPCs think, act, explain themselves, and hold money. NPCs are the heart of the "NPCs with wallets" requirement and the main reason the economy feels alive. Formulas referenced here live in `ECONOMY.md` §8; wallet mapping lives in `NEAR_INTEGRATION.md` §4.

---

## 1. Design goals

1. **Real economic actors.** Each NPC has its own wallet, inventory, budget, and decision rule. It trades through the same market rules as players. No NPC gets special pricing powers, except the Captain's demand rule.
2. **Explainable.** Every action carries a structured `Reasoning` object plus a one-line English string shown in the UI.
3. **Deterministic by default.** Heuristic rules + seeded RNG. Same seed → same behavior. The demo never depends on an LLM or network.
4. **Safe with money.** A **policy guard** sits between an NPC's decision and its wallet. The same guard concept maps onto NEAR function-call-key limits (what justifies the `security` track, if we claim it).
5. **Distinct personalities.** Small parameter differences (margin, patience, risk) create different behaviors without separate code per NPC.

## 2. The agent loop (each tick, per NPC)

```
observe(world, self)  →  decide(observation, personality, rng)  →  guard(actions, policy)  →  act(sim)  →  log(reasoning)
```

- **observe:** read-only snapshot: own cash/inventory, recent prices (last 2 days median and last trade) per good per venue, order-book top-of-book for venues the NPC can access, ship status, active events, own recent fills.
- **decide:** produce a list of **intended actions** (post ask, post bid, cancel, craft, produce, change venue, pause) each with a `Reasoning`.
- **guard:** enforce the policy (spend limits, caps, allowed goods). Rejected actions are logged with the rejection reason.
- **act:** the sim applies the actions through the same code path as player actions.
- **log:** append `NpcDecision` events (see §5).

NPCs act on a staggered schedule (e.g., `tick % period == offset`) so they do not all act on the same tick, which would create artificial synchrony. Default period: every tick for orders, once per day for strategy review.

## 3. Types

```ts
type Personality = {
  margin: number;          // target markup/markdown, e.g. 0.08–0.20
  riskTolerance: number;   // 0–1; higher = holds more inventory, accepts thinner margins
  patience: number;        // 0–1; higher = slower to change price after shocks
  venueLoyalty: number;    // 0–1; stickiness to the current venue vs. switching for cost
};

type Observation = {
  tick: number; day: number;
  self: { cash: Pips; inventory: Record<Good, number>; stallId: string };
  prices: Record<Good, { lastTrade?: Pips; medianTwoDays?: Pips; bestBid?: Pips; bestAsk?: Pips }>;
  venues: Venue[];                 // Harbor Market + halls with fee info
  ship: { docked: boolean; ticksToArrival: number; ceiling?: Pips };
  demand: Record<Good, { soldLast24t: number; listedSupply: number; avgListedSupply2d: number }>;
  events: ActiveEvent[];           // e.g., Storm
};

type Reasoning = {
  code: ReasonCode;                // machine-readable, e.g. 'LOW_STOCK_RAISE_ASK'
  factors: Record<string, number | string>; // inputs that drove the decision
  text: string;                    // human line: "Salt is scarce, raising my price 12%."
};

type NpcAction =
  | { kind: 'POST_ASK'; good: Good; qty: number; price: Pips; venue: VenueId }
  | { kind: 'POST_BID'; good: Good; qty: number; price: Pips; venue: VenueId }
  | { kind: 'CANCEL'; orderId: string }
  | { kind: 'PRODUCE'; good: Good; qty: number }      // producers only
  | { kind: 'CRAFT'; recipe: RecipeId }               // Cooper, Smokehouse
  | { kind: 'PAUSE'; until?: number };                // stop buying/producing
```

## 4. Archetypes

Each archetype is a small module under `packages/sim/src/npcs/<archetype>.ts` exporting `decide(obs, personality, rng): Decision[]`.

### 4.1 Fisherman (×3)
- **Produces** fish daily: `daily_output = base * rate_factor * weather * (1 - fatigue)` (ECONOMY §8.3).
- **Sells** fish via ask pricing (ECONOMY §8.1); sells down excess (stock > 2× target) at up to 15% discount because fish spoil.
- **Personality spread:** one cautious (margin 0.10, patience high), one aggressive (margin 0.18, low patience), one average.
- **Storm behavior:** catch collapses; remaining stock is priced up. Reasoning: *"Storm kept the boats in. Stock is low, raising my price 20%."*

### 4.2 Salt-panner (×2)
- **Produces** salt (non-perishable), holds higher inventory buffer, reacts to low stock by raising price (`k_stock`).
- **Personality:** one hoarder (high riskTolerance), one steady seller.
- Reasoning example: *"Salt is scarce, raising my price 12%."*

### 4.3 Woodcutter (×2)
- Steady low-margin timber supplier. Pauses production if price < 0.6× anchor for 2 consecutive days, resumes when above 0.75×.

### 4.4 Cooper (×1)
- **Buys** timber (bid formula ECONOMY §8.2), **crafts** barrels (3 timber → 1), **sells** barrels.
- Bid ceiling derived from the barrel's recent median. If barrels are not selling, reduce timber purchases and say so.
- Reasoning example: *"Barrel demand is up (Smokehouse + 2 players bought). Buying more timber."*

### 4.5 Smokehouse (×1)
- **Buys** fish, salt, barrels; **cures**; **sells** cured fish. Competes with players.
- Computes curing margin each tick from live prices; **pauses** when margin ≤ 0 or cash would drop below `cashFloor`.
- Chooses sale timing: holds cured fish until the ship is within 12 ticks of arrival, unless spoilage or cash needs push it to sell earlier.
- Reasoning example: *"Cured fish margin is only 3% at current fish prices. Pausing purchases until fish drops."*

### 4.6 Ship Captain (×1, visiting)
- Not a free-roaming strategist. Follows the deterministic buying rule in ECONOMY §7: buys cheapest asks (after fees) under the ceiling until the volume is met; adjusts the ceiling per visit.
- Reasoning on departure: *"Only 9 of 16 cured fish available. Raising my ceiling next visit to 66 marks."*
- Pays in **minted marks** (faucet). The mint is a logged event.

## 5. Reasoning log (UI + judging asset)

- Store an `NpcDecision` event per meaningful change (not per no-op tick): `{tick, npcId, actions[], reasoning[], guardResult}`.
- UI shows the last 5 decisions per NPC and a global feed (events and notable decisions, capped to avoid noise).
- Deduplicate: if the same reason code and rounded factors repeat, show once with a "×N" count.
- Reasoning strings are generated from templates keyed by `ReasonCode` with factor interpolation. Template list lives in `packages/sim/src/npcs/reasons.ts`. Add a template whenever you add a reason code.
- **Test:** every action emitted by any NPC must have a non-empty `reasoning.text` (property test).

Example reason codes: `LOW_STOCK_RAISE_ASK`, `EXCESS_STOCK_DISCOUNT`, `PRICE_BELOW_COST_PAUSE`, `MARGIN_NEGATIVE_PAUSE_BUY`, `STORM_SUPPLY_CUT`, `SHIP_NEAR_HOLD_STOCK`, `SWITCH_VENUE_LOWER_FEE`, `MERCHANT_CLOSED`, `MERCHANT_REOPENED`, `CAPTAIN_CEILING_RAISE`, `CAPTAIN_CEILING_LOWER`, `DEMAND_UP_RAISE_ASK`, `MARKET_GLUT_LOWER_BID`, `SHOP_CLOSED_BROKE`, `INVESTMENT_RECEIVED_REOPEN`, `RESCUED_BY_TOWN_FUND`.

### 5.1 Where the rules live: one readable file

All merchant pricing and quantity logic is in **`packages/sim/src/pricing/merchant-rules.ts`** (organizers' requirement: "merchant pricing rules live in one readable file"). Design it so a non-programmer judge can read it top to bottom:

```ts
export const MERCHANT_RULES: Record<Archetype, MerchantRule> = {
  salt_panner: {
    sells: ['salt'],
    targetStock: 14,
    ask:  { margin: 'personality.margin', kStock: 0.4, kDemand: 0.25, clamp: [0.5, 2.5] },
    produce: { base: 6, elasticity: 0.8, pauseBelowAnchorRatio: 0.6, resumeAboveAnchorRatio: 0.75 },
    // reasons are chosen by the first matching condition
    reasons: [
      { when: 'stock < 0.5 * target', code: 'LOW_STOCK_RAISE_ASK',  text: 'Salt is scarce, raising my price {pct}%.' },
      { when: 'demandPressure > 0.3', code: 'DEMAND_UP_RAISE_ASK',  text: 'Salt is selling fast, nudging my price up {pct}%.' },
      { when: 'stock > 2 * target',   code: 'EXCESS_STOCK_DISCOUNT', text: 'Too much salt on hand, discounting {pct}%.' },
    ],
  },
  // ...one block per archetype
};
```
Rules: comments explain each rule in plain English; no pricing logic outside this file; `explain(decision)` builds the tooltip text from the same table so UI text can never drift from behavior.

**Tooltip contract:** hovering (or tapping on mobile) a merchant's price or card shows the current reason text plus the key factors (stock vs. target, recent demand, margin, venue fees), e.g. *"Salt is scarce, raising my price 12%. Stock 3 of 14, sold 9 in the last day, margin 16%."*

## 6. Policy guard (spending limits and caps)

Every NPC has a `Policy`:

```ts
type Policy = {
  maxSpendPerDay: Pips;          // total purchases per day
  maxSpendPerTrade: Pips;
  cashFloor: Pips;               // never go below ~3 days of living costs
  allowedGoodsBuy: Good[];       // e.g., Fisherman buys nothing; Cooper buys timber only
  allowedGoodsSell: Good[];
  maxOrderQty: number;
  allowedVenues: VenueId[] | 'any';
};
```

- The guard is **pure code** in `packages/sim/src/npcs/guard.ts`, with unit tests. Rejections are logged.
- **Onchain mapping (if we pursue the `security` claim):** each NPC's sub-account is operated by the server using a **function-call access key** scoped to the goods contract's transfer methods with an allowance. The per-trade/day caps mirror the policy. This is real defense-in-depth, not just rhetoric. See `NEAR_INTEGRATION.md` §4.3. Do not claim this in the README unless it is implemented and demonstrable.

## 7. Personality seeds (starting points)

| NPC | margin | riskTolerance | patience | venueLoyalty |
| --- | --- | --- | --- | --- |
| Fisherman A (cautious) | 0.10 | 0.3 | 0.8 | 0.7 |
| Fisherman B (aggressive) | 0.18 | 0.7 | 0.2 | 0.3 |
| Fisherman C (average) | 0.14 | 0.5 | 0.5 | 0.5 |
| Salt-panner A (hoarder) | 0.16 | 0.8 | 0.7 | 0.6 |
| Salt-panner B (steady) | 0.12 | 0.4 | 0.5 | 0.5 |
| Woodcutter A/B | 0.08 / 0.10 | 0.3 / 0.4 | 0.6 / 0.5 | 0.6 |
| Cooper | 0.15 | 0.5 | 0.5 | 0.5 |
| Smokehouse | 0.18 | 0.6 | 0.4 | 0.4 |

Give each NPC a name and one-line bio for flavor (e.g., "Maren, Captain of the *Gull*"). Keep copy short.

## 8. Going broke, closing, and being reopened by an investor

Organizers' requirement: *if a merchant goes broke, it closes its shop until someone invests in it.* Full parameters are in `ECONOMY.md` §8.6.

- **Trigger:** cash below one day's living cost for 3 consecutive days and no sellable inventory.
- **Closed state (`CLOSED`):** cancel open orders; stop buying, selling, and producing; the NPC card shows **"Closed: needs investment"** with the minimum amount. Emit `MERCHANT_CLOSED` + reasoning `SHOP_CLOSED_BROKE` ("Out of cash after the storm. Closing until someone backs me.").
- **Reopen by investment:** a player invests 150 to 600 marks. The principal goes to the merchant's wallet and the NPC resumes with a fresh policy. The investor earns 25% of the merchant's positive daily net profit until payouts reach 125% of principal. Emit `MERCHANT_REOPENED` + reasoning `INVESTMENT_RECEIVED_REOPEN`.
- **Fallback rescue:** if nobody invests within 5 game days, the Town Fund seeds the merchant (up to 150 marks, no profit share). Emit `RESCUED_BY_TOWN_FUND`. This keeps the town alive when no players are around (for example, during judging or the time-lapse).
- **Time-lapse and headless runs** use scripted "investor bots" or the rescue fallback so closures resolve without a human.
- **Onchain:** the closed merchant keeps its sub-account. Investment and profit-share payouts are ordinary marks transfers settled in batches. Never mint for investments.
- **Always surface it:** closures, investments, and rescues go to the event feed; this is a gameplay moment, not an error.

## 9. Interaction with venues

- NPCs trade in the Harbor Market or Halls by effective cost (ECONOMY §6.3), weighted by `venueLoyalty` to avoid thrashing.
- NPCs **do not own Halls** in the MVP. Halls are the player-owned feature. (Optional stretch: an NPC "Rival Hall" with a fixed 3% fee to guarantee competition when no players are present. Label it clearly as NPC-run.)

## 10. Optional LLM advisor mode (stretch, off by default)

Purpose: make NPC strategy feel more agentic while keeping tick-level behavior deterministic.

- **Cadence:** once per game day per NPC, not per tick.
- **What it may change:** only `Personality` fields within bounded ranges (e.g., margin ±0.04, patience ±0.2) and a short `intent` string shown in the UI.
- **Contract:** the model receives a compact JSON digest of the NPC's last day (prices, fills, cash, stock) and must reply with strict JSON validated by a zod schema. On any error, timeout, or invalid output: **fall back silently** to the current personality.
- **Safety:** the model never emits raw orders or touches wallets. The guard and deterministic rules remain in charge. No secrets in prompts.
- **Determinism:** record each LLM response in the event log so replays are reproducible without calling the model.
- **Flag:** `NPC_LLM_ADVISOR=1`. Default off. The demo and tests must pass with it off.
- **Cost control:** hard cap on calls per run; log token usage.

## 11. Testing NPCs

- **Unit:** each archetype's `decide` with fixed observations → expected action kinds and reason codes.
- **Property:** all actions pass the guard or are rejected with a reason; every action has reasoning text; no action produces negative balances.
- **Scenario:** run 30 days with no players; assert no NPC is permanently insolvent without replacement, prices stay within bounds outside Storms (ECONOMY §11).
- **Storm:** run the Storm scenario and assert the documented response (see `TESTING_AND_DEMO.md` §4).
