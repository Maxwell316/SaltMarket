# BUILD_PLAN.md — Saltmarket

The ordered plan from empty repo to submission. **Deadline: 2026-11-14T23:59:00Z.** Plan assumes work starts on 2026-11-05. Target submission time: **by 2026-11-14 12:00 UTC**, leaving 12 hours of buffer. Update the checkboxes and the Status block in `CLAUDE.md` as you go.

**Ordering principle:** loop first, chain second. The game must be fun and legible on `MemoryLedger` before any contract work starts, because the track is *judged on the game loop, not the token*.

---

## Timeline at a glance

| Date (2026) | Phase | Goal |
| --- | --- | --- |
| Nov 5 | 0 | Scaffold repo, config, CI-ish scripts, docs in place |
| Nov 5–7 | 1 | Deterministic sim: goods, recipes, markets, tick loop, headless runs |
| Nov 7–8 | 2 | NPC agents + reasoning + economy tuning (headless) |
| Nov 8–9 | 3 | Server + web UI: playable loop in a browser (MemoryLedger) |
| Nov 9–11 | 4 | NEAR: contract, accounts, batched settlement, social login, Judge drawer |
| Nov 11–12 | 5 | Storm + events + Halls polish, balance pass, bug bash |
| Nov 12–13 | 6 | README, demo video, demo post draft, public repo prep |
| Nov 13–14 | 7 | Buffer, final checks, human-confirmed submission |

If you are behind at the end of any phase, apply the **cut list** (§ bottom) rather than compressing later phases.

---

## Phase 0 — Scaffolding (Nov 5)

**Tasks**
- [ ] **Gate:** present `docs/GAME_LOOP.md` to the human, get approval/edits, record `Loop approved: <date>` in `CLAUDE.md` Status. **No code before this.**
- [ ] Read the organizers' starting points (NEP-245, near-dapp skill, intents basics) and begin `NEAR_INTEGRATION.md` §10.
- [ ] Initialize pnpm monorepo per `CLAUDE.md` §5; TypeScript strict; ESLint + Prettier; Vitest.
- [ ] `.gitignore` includes `.env*`, `node_modules`, SQLite files, build outputs, `*.key`, `credentials/`.
- [ ] `.env.example` with names only.
- [ ] `packages/shared`: types, zod schemas, `config/economy.ts` per `ECONOMY.md` §13.
- [ ] Commands from `CLAUDE.md` §10 stubbed and runnable.
- [ ] Ask the human the open questions in `CLAUDE.md` §11–12.

**Acceptance:** `pnpm install && pnpm typecheck && pnpm test` pass on an essentially empty codebase.

## Phase 1 — Deterministic simulation (Nov 5–7)

**Tasks**
- [ ] Seeded RNG utility + tests.
- [ ] `WorldState`, accounts, inventories, supply tracking (genesis/minted/burned).
- [ ] Production + crafting queues; recipes from config.
- [ ] Gather nodes (cap, regeneration, cooldown, yield tiers, per-player daily cap).
- [ ] Spoilage with seeded stochastic rounding.
- [ ] Harbor Market (posted prices, one purchase per tick) and Hall order-book matching (price-time).
- [ ] Fees: harbor tax → Town Fund; hall fee → owner; self-trade prevention.
- [ ] Daily boundary: rent, living costs, upkeep, demurrage, Town Fund cap burn.
- [ ] Captain logic (arrival, docking, ceiling updates, minted payment).
- [ ] Event emission for everything; metrics rollup.
- [ ] Invariant checks (conservation, no negatives) as a debug assertion after each tick in dev.
- [ ] `pnpm sim:headless` printing a compact day-by-day table.

**Acceptance:** 30-day headless run with scripted dummy producers/consumers completes with all invariants passing; determinism test (same seed → same event-log hash) passes.

## Phase 2 — NPC agents and tuning (Nov 7–8)

**Tasks**
- [ ] `observe/decide/guard/act` loop; staggered scheduling.
- [ ] Six archetypes per `NPC_AGENTS.md`; personalities; reason codes + templates.
- [ ] **All merchant pricing in `pricing/merchant-rules.ts`** with `explain()`; stock + demand + glut terms; test that no price logic lives elsewhere.
- [ ] Policy guard + unit tests.
- [ ] Merchant closure, player investment with profit share, reopen, and Town Fund rescue fallback.
- [ ] Tuning pass per `ECONOMY.md` §11 across ≥10 seeds, NPC-only and with bot players; start `docs/TUNING_LOG.md`.
- [ ] Storm event implemented in the sim (supply shock) and a first `pnpm sim:storm`.

**Acceptance:** economy meets §11 stability checks; Storm produces a visible price spike, NPC reasoning changes, Captain ceiling rise, and recovery within ~5 game days; every NPC action has reasoning text.

## Phase 3 — Server + web UI (Nov 8–9)

**Tasks**
- [ ] Fastify server, tick loop with speed control, WS + REST per `ARCHITECTURE.md` §4.
- [ ] SQLite event log + snapshots; boot/restore.
- [ ] `GuestAuth` and session handling.
- [ ] Web client: Town view, Market panel with sparklines, Stall panel, NPC cards with reasoning, Event feed, Hall screen, Leaderboard.
- [ ] Onboarding (first-minute flow from `GAME_DESIGN.md` §3).
- [ ] Judge panel (Storm/Fair Winds/Salt Glut, speed, seed) gated by flags.
- [ ] Gather nodes UI; Invest action on closed merchants; reasoning tooltips on every merchant price and card.
- [ ] **"Watch a week" time-lapse** (sandbox clone, 168 ticks in about 60 s, frames every 6 ticks, optional scripted shocks) + `pnpm sim:week`.

**Acceptance:** a new person can open the app and complete buy → cure → sell to the Captain in under 3 minutes without instructions; no player-facing crypto vocabulary.

## Phase 4 — NEAR integration (Nov 9–11)

**Tasks**
- [ ] Verify docs and fill `NEAR_INTEGRATION.md` §10 *before coding*.
- [ ] Ask the human for the testnet authority account and key setup (never invent).
- [ ] `contracts/goods` NEP-245 with authority mint/burn, storage handling, pause; contract tests.
- [ ] Deploy to **testnet** (human-confirmed where required).
- [ ] `NearLedger` + settlement queue + idempotent batch IDs + reconciliation.
- [ ] NPC and Town Fund and Captain sub-accounts created.
- [ ] Social login adapter with `GuestAuth` fallback; account creation on first login.
- [ ] Judge Ledger drawer with explorer links and queue status.
- [ ] (Stretch) function-call keys for NPC accounts + over-limit rejection demo script.
- [ ] (Stretch) NEAR Intents cash-out demo in Judge panel.

**Acceptance:** after a Storm run, a reconciliation report shows sim balances equal ledger balances for all NPCs and a sample of players; killing the RPC mid-run does not stop the game and the queue drains after recovery.

## Phase 5 — Events, Halls, polish (Nov 11–12)

**Tasks**
- [ ] Fair Winds and Salt Glut events.
- [ ] Hall competition scenario: two halls with different fees; NPCs migrate volume; show fee income.
- [ ] Balance and legibility pass: tooltips explaining price moves, friendly errors, ▲/▼ indicators.
- [ ] Bug bash with the human; fix top issues only.
- [ ] Playwright smoke test (login → buy → cure → sell).

**Acceptance:** Storm demo is repeatable from a seed and looks good on screen; Hall competition visibly shifts volume.

## Phase 6 — Docs, demo, public repo prep (Nov 12–13)

**Tasks**
- [ ] Write `README.md` per `SUBMISSION.md` §2 (judge-readable, "do X → see Y").
- [ ] Record the demo video using the script in `TESTING_AND_DEMO.md` §5 (record early; do not wait until the last day).
- [ ] Draft the demo post text for the human to publish (the human posts it).
- [ ] Secret scan the repo and git history; add `LICENSE`; ensure `.env.example` has no values.
- [ ] Prepare the "how to run offline in one command" path and verify on a clean clone.

**Acceptance:** a fresh clone → `pnpm install && pnpm dev` → playable in under 2 minutes with no accounts or keys.

## Phase 7 — Buffer and submission (Nov 13–14)

**Tasks**
- [ ] Re-read the live rules: https://nearhackathon.com/HACKATHON.md (dates/rules can change).
- [ ] With the human: make the repo public (human confirms).
- [ ] Human publishes the demo post on X (says it was built at NEAR HACKATHON; attaches the video or screenshots) and gives you the link.
- [ ] Collect every submission field from the human (see `SUBMISSION.md` §3). Do not guess any.
- [ ] Print the full JSON payload; wait for the human's explicit "yes"; submit once.
- [ ] Save the confirmation (201). Do not commit the payload.

**Acceptance:** 201 response from the API, noted in `CLAUDE.md` Status.

---

## Cut list (apply in this order if behind schedule)

1. NEAR Intents cash-in/out.
2. LLM advisor mode.
3. Function-call keys and the `security` track claim.
4. Real social login (ship `GuestAuth` + honest note; keep the adapter).
5. Fair Winds / Salt Glut events (keep Storm).
6. Sparklines polish, leaderboard extras.
7. Stall upgrades.
8. Per-NPC onchain accounts → fall back to a single NPC pool account with an internal ledger *(last resort; weakens "NPCs with wallets", so avoid unless unavoidable and disclose it)*.

**Never cut:** the approved game-loop doc, the one-file merchant rules, reasoning tooltips, the "Watch a week" time-lapse (the track prompt requires it), merchant closure and investment, the Storm demo, NPC reasoning lines, the Exchange Hall, invariant tests, the judge-ready README, the offline-first fallback.

## Daily checkpoint ritual

At the end of each work session:
1. Run `pnpm typecheck && pnpm test`.
2. Update the Status block in `CLAUDE.md`.
3. Commit locally with a clear message.
4. Tell the human: what works now, what's next, and anything blocking.
