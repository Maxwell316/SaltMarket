# TRACK_BRIEF.md — Games With Real Economies (`games`)

The organizers' official track page, plus a compliance matrix mapping every requirement to where we satisfy it and how it is tested. **This is the scoring lens.** When in doubt about priority, re-read this file.

---

## 1. Official track page (from the organizers)

**Track id:** `games` · **Tagline:** The economy is the gameplay

> Build NPCs with wallets, player-owned markets and economies that evolve without a dev patch. Judges score the game loop, with a bonus if players never notice the chain.

**Lean on:** Agent NPCs, social login, NEAR Intents, NEP-245

**Start here (read these before building the related part; record findings in `NEAR_INTEGRATION.md` §10):**
- NEP-245 multi token standard: https://github.com/near/NEPs/blob/master/neps/nep-0245.md
- near-dapp agent skill: https://github.com/near/agent-skills/tree/main/skills/near-dapp
- What are intents: https://docs.near-intents.org/getting-started/what-are-intents

### Organizers' example prompt (verbatim)

```text
Track: Games With Real Economies (id: games).

Build [a market-town game]. Players gather materials and trade with NPC merchants. Each merchant is an agent with its own wallet and inventory, and sets prices from its stock and recent demand. If players flood the market with iron, iron gets cheap. If a merchant goes broke, it closes its shop until someone invests in it.

Requirements:
- Items are NEP-245 multi tokens, so one contract holds every item type.
- Players sign in with a social or email login. No seed phrases, no wallet popups, no gas prompts in the normal game loop.
- Merchant pricing rules live in one readable file. Show a merchant's reasoning in a tooltip.
- A fast-forward mode that simulates a week of trading in a minute, so judges can watch the economy move.

Design the game loop on paper first: what the player does every 30 seconds, every 5 minutes and every session. Show me that before any code.
```

Saltmarket is our answer to the `[a market-town game]` placeholder: a harbor town. The prompt's "iron" example maps to our goods (fish, salt, timber).

## 2. Compliance matrix

Status column is for Claude Code to maintain: `TODO` → `PARTIAL` → `DONE`, with a one-line note.

| # | Requirement (source) | How Saltmarket satisfies it | Specified in | Verified by | Status |
| --- | --- | --- | --- | --- | --- |
| R1 | Players **gather materials** and trade with NPC merchants | Shared gather nodes (Pier, Pans, Grove) with cooldowns and regeneration; players sell to NPC merchants and each other | `GAME_DESIGN.md` §4, `ECONOMY.md` §5 | Scenario test: bot gathers + sells | TODO |
| R2 | Each merchant is an agent with **own wallet and inventory** | NPC sub-accounts onchain; inventory in sim, mirrored by settlement | `NPC_AGENTS.md`, `NEAR_INTEGRATION.md` §4 | Reconciliation test; Ledger drawer shows NPC accounts | TODO |
| R3 | Merchant **sets prices from its stock and recent demand** | Ask/bid formulas use `stock_pressure` and `demand_pressure` | `ECONOMY.md` §8.1–8.2 | Unit tests on formulas | TODO |
| R4 | **Flooding** a good makes it cheap | Glut response lowers merchant bids and prices fall via competition and supply elasticity | `ECONOMY.md` §8.5 | Scenario: bots dump 200 timber, price falls ≥ 25% within a day | TODO |
| R5 | A **broke merchant closes** until someone **invests** | Closed state; player "Invest" action reopens it with a profit share; Town Fund rescue only as a long-delay fallback | `NPC_AGENTS.md` §8, `ECONOMY.md` §8.6 | Scenario test 3.4 | TODO |
| R6 | Items are **NEP-245 multi tokens**, one contract | `contracts/goods` holds marks, goods, and hall licenses as token ids | `NEAR_INTEGRATION.md` §2 | Contract tests | TODO |
| R7 | **Social or email login**; no seed phrases, wallet popups, or gas prompts in the normal loop | `AuthProvider` (social/email) + server-paid gas; `GuestAuth` only as fallback | `NEAR_INTEGRATION.md` §5–6 | E2E smoke; UI vocabulary lint | TODO |
| R8 | **Merchant pricing rules live in one readable file** | `packages/sim/src/pricing/merchant-rules.ts` is the only place merchant prices are decided; heavily commented | `ARCHITECTURE.md` §3.6, `NPC_AGENTS.md` §5.1 | Lint/test: no price logic outside the file | TODO |
| R9 | **Merchant reasoning in a tooltip** | Hover/tap any price or NPC card → tooltip generated from the same rules file | `NPC_AGENTS.md` §5, `GAME_DESIGN.md` §11 | E2E: tooltip text present and non-empty | TODO |
| R10 | **Fast-forward**: a week of trading in a minute | "Watch a week" time-lapse sandbox: 168 ticks streamed in ~60 s, does not touch the live world | `ARCHITECTURE.md` §3.5, `GAME_LOOP.md` §6 | Test: 168 ticks complete and stream within 60–70 s wall-clock | TODO |
| R11 | **Game loop designed on paper first**: every 30 s, 5 min, session; shown before any code | `docs/GAME_LOOP.md`; Claude Code must present it and get the human's OK before Phase 0 code | `GAME_LOOP.md`, `CLAUDE.md` §3 | Human approval recorded in `CLAUDE.md` Status | TODO |
| R12 | Player-owned markets | Exchange Hall licenses with owner-set fees | `GAME_DESIGN.md` §5, `ECONOMY.md` §6 | Scenario 3.3 | TODO |
| R13 | Economy evolves without a dev patch | Sinks/faucets, elasticity, Storm response | `ECONOMY.md`, `TESTING_AND_DEMO.md` §4 | Stability + Storm tests | TODO |
| R14 | Bonus: players never notice the chain | Vocabulary rules, optimistic UI, hidden Judge/Ledger drawer | `GAME_DESIGN.md` §9 | UI vocabulary lint, review | TODO |
| R15 | Lean on NEAR Intents (optional) | Cash-out demo in Judge panel only if time allows | `NEAR_INTEGRATION.md` §8 | Manual demo | TODO (optional) |

## 3. Judging implications

- The score is on the **game loop**. R11 and R1 (what the player actually does) matter more than any contract detail.
- R8–R10 are **judge-facing affordances**: readable rules, tooltips, and the time-lapse exist so a judge can *see* the economy reason and move without playing for an hour. Treat them as first-class features, not polish.
- R5 (closing and investing) gives players a meaningful, non-trivial economic role beyond trading. It is a strong differentiator; keep it in the MVP.

## 4. Process rule from the organizers

"Show me the loop before any code." Claude Code must:
1. Open `docs/GAME_LOOP.md` and present its cadence summary to the human.
2. Ask for explicit approval or edits.
3. Record "Loop approved on <date>" in `CLAUDE.md` Status.
4. Only then begin Phase 0 scaffolding.
