# NEAR_INTEGRATION.md — Saltmarket

How Saltmarket uses NEAR, kept deliberately **invisible** to players. This document describes the *intended* design and lists what must be **verified in official docs before implementation**. Do not treat any method name, package version, or address here as confirmed until it appears in the "Verified facts" table (§10).

---

## 0. Ground rules

1. **Verify first.** Before implementing each section, read the current official docs (https://docs.near.org/) and record the verified facts in §10 with links and dates. Names below (e.g., `mt_batch_transfer`) are from the standards as we understand them and must be confirmed.
2. **Testnet by default.** All development uses NEAR testnet. Mainnet is only for the human-supplied prize payout account, and any mainnet signing requires explicit human confirmation (see `CLAUDE.md` §3).
3. **Never invent accounts, contract IDs, or keys.** The human supplies the authority account. You may propose names for sub-accounts under it (e.g., `fisher-1.<authority>`), but creating accounts on a network requires the human's approval and key setup.
4. **The game never waits on the chain.** If NEAR is slow or down, the game runs on `MemoryLedger` and the settlement queue catches up later.
5. **No keys in the repo.** Env vars only.

### Organizers' starting points (read first, then record findings in §10)

- NEP-245 multi token standard: https://github.com/near/NEPs/blob/master/neps/nep-0245.md
- near-dapp agent skill: https://github.com/near/agent-skills/tree/main/skills/near-dapp (guidance for building NEAR dapps; read it before writing contract or client integration code)
- What are intents: https://docs.near-intents.org/getting-started/what-are-intents

## 1. What goes onchain vs. off-chain

| Concern | Where | Why |
| --- | --- | --- |
| Matching, pricing, NPC decisions, ticks | Off-chain (sim) | Speed, determinism, cost |
| Balances of marks and goods | Onchain (NEP-245 contract), settled in batches | Real ownership; verifiable |
| Hall licenses (ownership of a venue) | Onchain (token id `hall_license`) | Player-owned markets |
| Account identity for players | Onchain account created behind social login | "Players own things" without crypto UX |
| NPC wallets | Onchain sub-accounts | "NPCs with wallets" |
| Fees and tax flows | Settled as transfers to Hall owner / Town Fund accounts | Visible fee income |
| Event feed, price history | Off-chain (SQLite) | Not needed onchain |

Be honest about this split in the README: **off-chain matching with onchain settlement and ownership**, batched.

## 2. Contract: `goods` (NEP-245 multi-token)

One Rust contract (`contracts/goods`) implementing the multi-token standard (NEP-245), holding all game assets as token ids:

| Token id | Meaning |
| --- | --- |
| `marks` | Currency (pips as the smallest unit) |
| `fish`, `salt`, `timber`, `barrel`, `cured_fish` | Goods |
| `hall_license` | Venue right (supply controlled by the authority; one per Hall) |

### 2.1 Required functionality
- **Standard NEP-245 methods** for transfer, batch transfer, balance queries, and token metadata (verify exact names and signatures in the standard and in `near-sdk-rs` docs).
- **Authority-controlled mint and burn** for the game server account (`mint(account, token_id, amount)`, `burn(account, token_id, amount)`): needed for genesis, production, captain payments, spoilage, and sinks. Restricted to an `authority` account set at init.
- **Batch settlement helper (optional):** a single method that applies many net deltas atomically (`settle(batch)`) to reduce transaction count. If the standard `mt_batch_transfer` plus mint/burn is sufficient, skip this.
- **Owner/authority rotation** and a pause switch (for demo safety).
- **Storage:** handle storage deposits properly (verify the registration/storage-management pattern for the multi-token standard and for `near-sdk-rs`). Pre-pay storage from the authority/relayer for player and NPC accounts so players never see a deposit prompt.

### 2.2 Non-goals
- No marketplace logic in the contract. No AMM. No governance. No tokenomics beyond the above.

### 2.3 Tests
- `near-workspaces` (or the current recommended framework; verify) tests: genesis mint, batch transfer, authority-only mint/burn, rejection of non-authority mint, balance queries, storage deposit flow.
- Property: sum of balances per token = minted − burned.

## 3. Account topology

```
<authority>.testnet                      # human-created; owns the goods contract + acts as game authority
├── goods.<authority>.testnet            # (or the authority itself hosts the contract; decide with human)
├── townfund.<authority>.testnet         # Town Fund
├── captain.<authority>.testnet          # Ship Captain treasury (mint recipient/spender)
├── npc-fisher-1.<authority>.testnet     # NPC sub-accounts (one per NPC)
├── npc-fisher-2.<authority>.testnet
├── ...
└── p-<shortid>.<authority>.testnet      # player accounts (created at first login)
```

- **Sub-accounts** make NPC wallets real, cheap to create, and clearly grouped under the game's authority.
- Naming must satisfy NEAR account id rules (lowercase alphanumerics with `-`/`_` separators, length limits; verify). The hackathon's `nearAddress` regex for the payout account is in `SUBMISSION.md`; payout is a **separate mainnet account chosen by the human**.
- Account creation is a server-side admin flow behind `LedgerAdapter.ensureAccount`.

## 4. NPC wallets and spending limits

### 4.1 Wallet per NPC
Each NPC has its own sub-account; its balances are mirrored via batched settlement. The Judge drawer links each NPC to its account on a testnet explorer (verify the current explorer URL).

### 4.2 Operation model
The server operates NPC sub-accounts. Two options; pick the simplest that works:
- **A. Settlement-only (MVP):** NPC balances change only through authority-signed batch settlements; NPC "keys" are not used for trades. Simple and robust.
- **B. Function-call keys (stretch, enables the `security` track):** each NPC sub-account gets a **function-call access key** restricted to the goods contract and specific methods with a gas allowance, used by the server to execute that NPC's own transfers. Combined with the sim's policy guard, this is defense-in-depth ("give AI money without giving it the keys to everything").

### 4.3 What to claim publicly
Only claim spending limits, function-call keys, or sub-account isolation in the README **if implemented and demonstrable**. A demonstrable version: a script that attempts an over-limit NPC transfer and shows it rejected (by guard and/or by key permissions).

## 5. Social login (players never see a wallet)

### 5.1 Requirements
- One-click sign-in with a **social or email** identity (the track requires social or email login; **no seed phrases, no wallet popups, no gas prompts** in the normal game loop).
- On first login, a NEAR account is created or derived and funded for storage; the player never sees seed phrases, addresses, or approval pop-ups.
- Server maps session → `playerId` → ledger account.

### 5.2 Implementation approach
- Define an `AuthProvider` interface in `apps/server`:
  ```ts
  interface AuthProvider {
    begin(req): Promise<LoginChallenge | Session>;
    complete(req): Promise<Session>;       // returns { playerId, displayName }
  }
  ```
- Implementations: `GuestAuth` (anonymous session, instant, for dev and the guaranteed demo fallback) and `SocialLoginAuth` (real provider).
- **Provider choice is open:** research current NEAR-ecosystem options for social/embedded-wallet login (check https://docs.near.org/ and the hackathon's ecosystem list in `HACKATHON.md`), pick the one with the least integration risk, and record it in §10. Ask the human before adding paid services or creating third-party accounts.
- **Custody model (be honest):** for the MVP, the server may custody player ledger accounts on their behalf ("embedded/custodial wallet"). State this plainly in the README. Do not claim self-custody unless implemented.

## 6. Gasless play (relayer)

- Players never pay gas. All transactions are paid by the **relayer/authority**.
- **MVP:** since the server submits settlements itself, gas is naturally server-paid; no player signing needed.
- **Stretch:** meta transactions (delegate actions, NEP-366; verify) so a player-held or provider-held key can sign an intent that the relayer submits. Only do this if social login provides a signer and time allows.
- Relayer account funding: the human funds testnet accounts via the official testnet faucet/tools; do not automate scraping faucets.
- Guard the relayer: rate limits per player, per-day gas budget, and an allowlist of contract methods.

## 7. Settlement details

- Input: net deltas per `(account, token)` for the window (see `ARCHITECTURE.md` §5.1).
- Build batches that respect gas and size limits (verify current limits; split into multiple transactions when needed).
- **Idempotency:** each batch has a deterministic `batchId` (`tickFrom-tickTo`); store it in the settlements table, and include it in the call (e.g., as a memo or via a contract-side "last settled batch" guard) so retries cannot double-apply. Design this explicitly; it is the main correctness risk.
- **Ordering:** apply burns/transfers/mints such that no intermediate negative balance occurs (net deltas make this easy; apply mints first, then transfers, then burns).
- **Reconciliation:** daily job compares sim vs ledger balances; mismatches are logged and surfaced in the Judge panel.

## 8. Optional: NEAR Intents cash-in/cash-out

- Listed in the track hints; **optional and last priority**.
- If attempted: read https://docs.near-intents.org/ first; scope it to a single "cash out marks → stablecoin" or "cash in" demo flow in the Judge panel only. The core loop must never depend on it.
- Skip entirely if Phase 4 is behind schedule (see `BUILD_PLAN.md` cut list).

## 9. Security checklist

- [ ] No secrets in git history, logs, client bundle, or error messages.
- [ ] Authority and relayer keys only from env; `.env*` in `.gitignore`.
- [ ] Contract mint/burn restricted to authority; tests prove non-authority calls fail.
- [ ] Relayer rate-limited and method-allowlisted.
- [ ] Batch idempotency verified by a retry test.
- [ ] Pause switch tested.
- [ ] Mainnet signing never occurs without explicit human confirmation.
- [ ] README honestly states custody model and the off-chain-matching/onchain-settlement split.

## 10. Verified facts (fill in as you verify; keep dates)

| Topic | What was verified | Source URL | Date |
| --- | --- | --- | --- |
| NEP-245 method names/signatures | _TBD_ | | |
| `near-sdk-rs` version and multi-token support/helper crates | _TBD_ | | |
| `near-api-js` current package name/API | _TBD_ | | |
| Storage management pattern for multi-tokens | _TBD_ | | |
| Sub-account creation rules/costs | _TBD_ | | |
| Function-call access keys (allowance, method scoping) | _TBD_ | | |
| Meta transactions / delegate actions (NEP-366) | _TBD_ | | |
| Social/embedded login option chosen + integration steps | _TBD_ | | |
| Testnet RPC endpoint and explorer URL | _TBD_ | | |
| Gas/size limits for batch calls | _TBD_ | | |

## 11. Failure and fallback matrix

| Failure | Behavior |
| --- | --- |
| RPC down or slow | Game continues on `MemoryLedger`; queue retries with backoff; status shows "ledger: queued N" in Judge panel |
| Batch tx fails (gas/size) | Split batch; retry; log |
| Social login provider fails | Fall back to `GuestAuth` (feature flag) |
| Account creation fails | Player plays with a pending ledger account; created lazily at next settlement |
| Contract bug found late | Pause switch; fall back to `MemoryLedger` for the demo; document honestly |
