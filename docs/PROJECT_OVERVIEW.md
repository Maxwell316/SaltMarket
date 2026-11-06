# PROJECT_OVERVIEW.md — Saltmarket

Purpose: give Claude Code (and any human reviewer) the full "why" behind the project, so design decisions stay consistent. Pair this with `CLAUDE.md`.

---

## 1. One-paragraph pitch

Saltmarket is a cozy harbor-town trading game where **the economy is the gameplay**. You run a stall, buy fish, salt, and timber, cure fish for ship captains, and compete or cooperate with a cast of NPC merchants who each hold their own wallet and explain their own pricing decisions. Players can buy an **Exchange Hall** and take a fee on every trade routed through it. Nobody patches the economy: prices, production, and strategies shift on their own, and a **Storm** event lets anyone watch the whole town adapt. NEAR quietly handles ownership and settlement; players sign in with a social login and press "Buy".

## 2. The hackathon brief we are answering

**Track:** Games With Real Economies (`games`).

> "Build NPCs with wallets, player-owned markets and economies that evolve without a dev patch. Judged on the game loop, not the token — bonus if players never notice the chain."

NEAR's suggested hints for this track: **AGENT NPCS, SOCIAL LOGIN, NEAR INTENTS, NEP-245.**

The organizers also published a full track page with an example prompt and explicit requirements (gathering, merchants that close until someone invests, one readable pricing-rules file, tooltips, a week-in-a-minute fast-forward, and a loop designed on paper first). It is reproduced with a compliance matrix in `TRACK_BRIEF.md`; the loop design is in `GAME_LOOP.md`.

### Brief → feature mapping

| Brief phrase | What it means for us | Where it lives |
| --- | --- | --- |
| "The economy is the gameplay" | Core loop is produce → trade → reinvest; no non-economic activity | `GAME_DESIGN.md` §2 |
| "NPCs with wallets" | Each NPC has its own onchain sub-account and balance, and acts as a real market participant | `NPC_AGENTS.md`, `NEAR_INTEGRATION.md` §4 |
| "Player-owned markets" | Exchange Halls with owner-set fees and rival halls | `GAME_DESIGN.md` §5, `ECONOMY.md` §6 |
| "Economies that evolve without a dev patch" | Feedback loops, self-balancing sinks/faucets, no hand-set prices | `ECONOMY.md` |
| "Judged on the game loop, not the token" | No token marketing; the loop must be clear and fun first | `BUILD_PLAN.md` phase order |
| "Bonus if players never notice the chain" | Social login, gasless relayer, no crypto vocabulary in UI | `NEAR_INTEGRATION.md` §5–6 |

## 3. How we expect to be judged (our interpretation)

Judges (humans and AI) will likely ask:

1. **Is the loop understandable in about a minute?** Can I tell what to do and why it's interesting?
2. **Is the economy alive?** Do prices move for visible reasons? Do NPCs behave like agents with a rationale?
3. **Does it survive a shock?** (Storm demo.) Does the system stabilize without developer intervention?
4. **Is the chain genuinely used but invisible?** Are balances and ownership onchain and verifiable, without crypto friction for players?
5. **Is it reproducible?** Can a judge (or an AI judge reading the README) run it and see the claimed behavior?
6. **Is it a real NEAR use?** Sub-accounts for NPCs, NEP-245 for goods, social login, gasless transactions.

**Design implication:** invest in *legibility* (clear UI, reasoning lines, price history, event feed) more than in content volume. A small readable world beats a big opaque one.

## 4. Success criteria

**Must-have (MVP, the submission is weak without these):**
- [ ] Playable browser loop: buy raw goods, craft cured fish, sell to NPCs/players, reinvest.
- [ ] 10 NPCs with wallets, budgets, decision rules, and visible reasoning.
- [ ] A ship captain NPC acting as the main demand source.
- [ ] Exchange Hall: buy a license, set a fee, earn fees; a second hall can compete.
- [ ] Storm event with a clear, repeatable town response.
- [ ] Sinks and faucets that keep the economy from collapsing or exploding over a 30-day headless run.
- [ ] NEAR testnet: NEP-245 contract, NPC and player balances mirrored via batched settlement, social-login accounts, relayer-paid gas.
- [ ] Judge-ready README and demo video.
- [ ] Players gather materials from shared nodes and trade with merchants.
- [ ] Merchants close when broke and can be reopened by a player's investment.
- [ ] All merchant pricing in one readable file, with reasoning tooltips in the UI.
- [ ] "Watch a week" time-lapse (7 game days in about 60 s).
- [ ] `GAME_LOOP.md` approved by the human before code.

**Should-have:**
- [ ] Price history sparklines and an event feed.
- [ ] "Ledger" drawer for judges with explorer links.
- [ ] Function-call-key spending limits on NPC wallets (this is what justifies the `security` track, if we want it).
- [ ] Deterministic seeded replays (`pnpm sim:storm --seed 42`).

**Nice-to-have (cut first):**
- [ ] LLM "advisor" mode for NPC strategy tuning.
- [ ] NEAR Intents cash-in/cash-out.
- [ ] Multiplayer presence (seeing other players' stalls move).
- [ ] Sound, animation polish.

## 5. Scope boundaries

**In scope:** a single town, 5 goods, one currency (marks), 10 NPCs, up to N concurrent players (target 20+ comfortably), 3 player roles (Stall Keeper, Hall Owner, Investor), 1 headline event (Storm) plus 2 minor events.

**Out of scope (do not build, even if tempting):**
- A custom token launch, tokenomics, or any speculative asset. The game currency is "marks" and exists to make the loop work.
- Combat, quests, narrative, character customization.
- Cross-game asset interoperability.
- Real-money cash-out as a core requirement (optional via NEAR Intents if time allows).
- Mobile-native apps (responsive web is enough).

## 6. Tracks to submit under

- **`games`** — primary and clearly fitting.
- **`agents`** (Agent Economy) — add if NPCs genuinely earn, spend, and pay each other onchain from their own wallets.
- **`swarms`** (AI Swarms) — add if NPC-to-NPC trade creates emergent price dynamics (it should, by design).
- **`security`** — add **only** if function-call-key spending limits on NPC wallets are actually implemented and demonstrable.

Judges can tell when a track is tacked on. List only tracks whose features exist in the repo and are named in the README.

## 7. Risk register

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Time sunk into contracts before the loop is fun | High | High | Phase order: sim and UI first, chain second. `MemoryLedger` runs the whole game |
| Economy collapses or inflates | High | High | Headless sim + conservation property tests + tuning pass before UI polish |
| NEAR integration details differ from assumptions | Medium | High | Verify-in-docs rule; adapter seam; fallback to batched settlement and `MemoryLedger` |
| Social login integration takes too long | Medium | Medium | Adapter interface; fallback to server-created accounts keyed to a session, with social login layered later |
| Testnet flakiness during demo | Medium | High | Record the demo video early; demo works offline with `MemoryLedger`; settlement queue retries |
| NPCs feel random, not agentic | Medium | High | Structured reasoning objects + readable English per decision |
| README unreadable to AI judges | Low | High | Follow `SUBMISSION.md` template: "do X → see Y" |
| Running out of time near deadline | High | High | Cut list in `BUILD_PLAN.md`; target submission 12 hours before the deadline |

## 8. Glossary

| Term | Meaning |
| --- | --- |
| Mark | In-game currency, integer minor units (1 mark = 100 pips) |
| Pip | Smallest currency unit |
| Tick | One game hour; the sim's unit of time |
| Day | 24 ticks |
| Stall | A player's (or NPC's) shop and inventory holder; costs daily rent |
| Harbor Market | The default free venue with posted prices and friction |
| Exchange Hall | Player-owned venue with order-book matching and an owner-set fee |
| Faucet | Anything that creates marks or goods |
| Sink | Anything that destroys marks or goods |
| Town Fund | Account that collects harbor tax and funds new-NPC seed capital and relief |
| Reasoning line | The short English explanation attached to every NPC action |
| Ledger | The abstraction over MemoryLedger / NearLedger holding balances and ownership |
| Judge mode | Opt-in UI drawer exposing onchain settlements for verification |
