# TESTING_AND_DEMO.md — Saltmarket

How we prove the economy works and how we show it off. The demo is a deliverable: the track is judged on the game loop, so the Storm scenario is tested like a feature.

---

## 1. Test pyramid

| Level | Tool | What it covers | Run |
| --- | --- | --- | --- |
| Unit | Vitest | Formulas, matching, fees, spoilage, guard, reason templates | every change |
| Property | Vitest + `fast-check` (or hand-rolled seeded loops) | Conservation, non-negative balances, determinism, reasoning present | every change |
| Scenario | Vitest + headless sim | 30-day stability, Storm response, Hall competition | before each phase ends |
| Contract | `near-workspaces` / `cargo test` (verify current recommended tooling) | Mint/burn authority, transfers, storage, pause | on contract change |
| Integration | Vitest + `MemoryLedger`/`NearLedger` | Settlement batching, idempotent retries, reconciliation | Phase 4+ |
| E2E smoke | Playwright | Login → buy → cure → sell to Captain; Judge panel Storm button | before submission |

## 2. Must-pass invariants (automate all)

From `ECONOMY.md` §10:
1. Mark conservation every tick: `Σ marks = genesis + minted − burned`.
2. Good conservation per good.
3. No negative balances or inventories.
4. Determinism: same seed + inputs → identical event-log hash after N ticks.
5. Fee identity per trade: `buyer_paid = seller_received + tax + fee`.
6. Hall fee ∈ [0, 5%].
7. Captain never buys above ceiling.
8. Every NPC action has non-empty reasoning text.
9. Guard: no NPC action violates its policy.
10. Settlement idempotency: replaying a batch ID never double-applies.
11. Reconciliation: sim and ledger balances match after a settlement flush.

Put invariants in a reusable `assertInvariants(state)` called after every tick in dev/test builds.

## 3. Scenario tests

### 3.1 Stability (NPC-only)
- 10 seeds × 30 days. Assert the checks in `ECONOMY.md` §11 (burn/mint 0.85–1.10 after day 5, price clamp dwell < 3 days outside events, supply within ±30%, Captain ceiling not pinned).

### 3.2 Stability with bot players
- 5 scripted bots (naive buy-low/sell-high, a curer, a hoarder, an idle player, a Hall owner). Assert no invariant breaks and no bot can generate unbounded profit (e.g., net worth growth cap over 30 days relative to the best NPC).

### 3.3 Hall competition
- Two Halls, fees 4% and 1%. Assert NPC volume share shifts toward the cheaper Hall within 3 days, and the higher-fee owner's income drops unless they cut fees.

### 3.4 Merchant closure, investment, and rescue
- Force an NPC to run out of cash. Assert it goes CLOSED with `MERCHANT_CLOSED` in the feed. Case A: a bot invests 200 marks → merchant reopens, investor receives 25% of positive daily profit until 125% of principal, conservation intact. Case B: nobody invests → Town Fund rescue after 5 days with `MERCHANT_RESCUED`. A closed merchant must never buy, sell, or produce.

### 3.5 Flood test (market glut)
Scripted bots gather and dump 200 timber on the market in one game day. Assert: timber median falls ≥ 25% within 1 day, merchants' bids drop (glut term), woodcutters reduce output, and prices recover to within ±20% of baseline in 3–5 days. Reasoning text for `MARKET_GLUT_LOWER_BID` appears.

### 3.6 Time-lapse ("Watch a week")
Assert: 168 ticks complete in the sandbox and frames stream over about 60 s (accept 55–75 s); the live world's state hash and event log are unchanged by a time-lapse; no ledger calls are made; invariants hold throughout; scripted shocks (storm day 3, closure day 4) occur on the right days.

### 3.7 One-file rules and tooltips
Static test: no merchant price computation outside `pricing/merchant-rules.ts`. E2E: hovering any merchant price shows non-empty reasoning text with key factors.

## 4. The Storm scenario (headline demo, also a test)

**Trigger:** at day 10 (seed 42), fish catch factor drops to 0.15 for day 10 and 0.75 for day 11, then returns to normal.

**Expected response (assert ranges, not exact values):**

| Time after Storm start | Expected behavior |
| --- | --- |
| Tick +1 to +6 | Fishermen's reasoning shows storm/low-stock; fish asks rise ≥ 15% |
| Day +1 | Median fish price up 30–80% vs. pre-storm baseline; Smokehouse reasoning shows curing margin squeezed; it pauses or reduces fish purchases |
| Day +1 to +2 | Cured fish supply thins; at next ship visit the Captain's fill is below 100%; his ceiling rises (reasoning shown) |
| Day +2 | Player curers with fish stockpiles see higher margins; Hall volume in fish rises; hall owner fee income up |
| Day +3 to +5 | Fishermen resume normal output; fish price decays toward the baseline; Smokehouse resumes; Captain ceiling begins to relax |
| By day +6 | Fish median within ±20% of the pre-storm baseline; no permanent bankruptcies beyond what replacement handles; invariants hold throughout |

**Report:** `pnpm sim:storm --seed 42` prints a table of price, volume, Captain ceiling, and key NPC reasoning for days −2 to +6, and exits non-zero if expectations fail.

**Judge-panel behavior:** the Storm button triggers the same event live; the UI highlights the event in the feed and shows ▲ arrows with tooltips ("Fish ▲ 42%: storm cut the catch").

## 5. Demo script (target 90–120 seconds; record early)

Prepare: fresh world at seed 42, speed set so a day ≈ 20–30 seconds for the video, Judge panel open, two Halls pre-created (one by the demo player, one by a second account) with different fees, a few prior trades so charts have history.

| Time | Action | What the viewer should see |
| --- | --- | --- |
| 0:00 | One-line intro over the Town view | "A harbor town where the economy is the game. Every merchant is an agent with a wallet." |
| 0:10 | Sign in with one click | No wallet UI; straight into the town |
| 0:15 | Buy fish and salt, craft/cure | Plain buttons, friendly confirmations |
| 0:30 | Click an NPC card | Reasoning line: "Salt is scarce, raising my price 12%." |
| 0:40 | Sell cured fish to the docked ship | Marks go up; trade shows in feed |
| 0:50 | Show the player-owned Hall and its fee income | Hall owner's cut from real trades |
| 1:00 | **Press Storm** | Event banner; fish ▲; NPC reasoning lines change; Captain ceiling rises |
| 1:25 | Show the town recovering | Prices relax, Smokehouse resumes, feed explains each step |
| 1:40 | Open the **Ledger drawer** (Judge mode) | Settlements with explorer links: "this was onchain the whole time" |
| 1:50 | Close with the line | "No patch. No tweaks. The town adapted on its own." |

**Backup plan:** if testnet is flaky during recording, record with `MemoryLedger` for the gameplay segments and record the Ledger drawer segment separately against testnet. Never fake onchain results; say so in the README if any part is recorded with `MemoryLedger`.

## 6. Judge reproduction steps (mirrors the README)

1. `pnpm install && pnpm dev` → open the URL.
2. Click **Play** (guest or social login).
3. Buy 6 fish and 3 salt, buy or craft 1 barrel, press **Cure** → see 4 cured fish after 6 ticks.
4. Wait for the ship (countdown in the top bar) and sell → see marks increase.
5. Click any NPC card → see its reasoning and recent decisions.
6. Open `?judge=1` → press **Storm** → watch fish prices and NPC reasoning change, then recover.
7. Open the Ledger drawer → see batched settlements and links (if `LEDGER=near`).
8. Click **Watch a week** → see 7 game days of trading in about a minute (prices, merchants, Captain ceiling) without changing the live town. Add a storm or close a merchant with the toggles and watch the response.
9. Hover any merchant's price → read its reasoning tooltip. Open `packages/sim/src/pricing/merchant-rules.ts` → see every merchant's pricing rules in one file.

Verify these steps on a **clean clone** before submission.

## 7. Demo checklist (day of recording and day of submission)

- [ ] Fresh DB, seed 42, `JUDGE_MODE=1`.
- [ ] Browser zoom and window size set for readability; sound muted.
- [ ] Testnet accounts funded; RPC reachable; queue empty at start.
- [ ] Ledger drawer shows recent settlements.
- [ ] No secrets visible on screen (env vars, keys, terminal history).
- [ ] Screenshots captured: Town view, Storm moment, NPC reasoning, Hall income, Ledger drawer.
- [ ] Video under ~2 minutes; captions on key moments.

## 8. Quality bar for "done"

- Typecheck, lint, unit/property/scenario tests green.
- Storm scenario passes and looks good on screen.
- E2E smoke green.
- Clean-clone run works.
- No flaky tests: if a test depends on timing, make it tick-based, not wall-clock.
