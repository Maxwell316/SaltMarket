# SUBMISSION.md — Saltmarket

Everything needed to prepare and send the NEAR HACKATHON submission correctly. This summarizes the organizers' `HACKATHON.md`. **The live file is authoritative:** https://nearhackathon.com/HACKATHON.md. Re-read it before submitting; dates and rules may change.

---

## 1. Rules recap

| Item | Value |
| --- | --- |
| Hackathon starts | 2026-11-01T00:00:00Z |
| **Development ends / submission deadline** | **2026-11-14T23:59:00Z** |
| Judging starts | 2026-11-16T00:00:00Z |
| Results announced by | 2026-11-18T23:59:00Z |
| Submission form (alternative) | https://nearhackathon.com/#apply |
| Submission API | `POST https://nearhackathon.com/api/applications` |
| One submission per team; **cannot be edited** | Send once, when the repo is public and the demo post is up |
| Outside the window the API answers 403 | Tell the human; don't retry |

**Required before submitting:**
1. Repository is **public** (by the deadline). Judges cannot open private repos.
2. The human has published a **public demo post on X** saying the project was built at NEAR HACKATHON, with the demo video or screenshots attached.
3. The human has confirmed the exact payload.

**You may:** draft the README, the demo post text, and the payload.
**You may not (without explicit human confirmation):** send the submission, post to X, make the repo public, push, sign transactions, deploy to mainnet.

## 2. README requirements (judges include AI)

The README is read by humans **and AI judges**. Write for a machine reader: plain, structured, concrete, verifiable. No marketing language, no hype.

### 2.1 Required README structure
1. **Title and one-sentence summary:** what Saltmarket is.
2. **Problem and idea:** why economy-as-gameplay with agent NPCs; two or three sentences.
3. **What you can do (features):** bullets that map 1:1 to the track brief:
   - NPCs with wallets (list the six archetypes and where their onchain accounts show up)
   - Player-owned markets (Exchange Hall)
   - Economy that evolves without a patch (sinks/faucets, Storm demo)
   - Chain invisible to players (social or email login, gasless, no crypto vocabulary in UI)
   - Players gather materials; merchants price from stock and demand; broke merchants close until a player invests
   - Merchant pricing rules in one readable file (link `packages/sim/src/pricing/merchant-rules.ts`) with reasoning tooltips
   - "Watch a week" fast-forward: a week of trading in about a minute
4. **How NEAR is used:** NEP-245 goods contract, sub-accounts for NPCs, social login, relayer-paid gas, batched settlement. State honestly: *off-chain matching, onchain settlement and ownership*; custody model; what is testnet vs. mainnet.
5. **Try it in 60 seconds (do X → see Y):** numbered steps from `TESTING_AND_DEMO.md` §6, one command to run, expected result per step.
6. **Verify the economy yourself:** `pnpm sim:headless --seed 42 --days 30` and `pnpm sim:storm --seed 42`; what the output should show.
7. **Verify onchain:** how to open the Ledger drawer (`?judge=1`), contract id, explorer link, sample tx hashes (testnet).
8. **Architecture in brief:** a short diagram and links to `docs/ARCHITECTURE.md`, `docs/ECONOMY.md`, `docs/NPC_AGENTS.md`, `docs/NEAR_INTEGRATION.md`.
9. **Tracks:** list the tracks entered and, per track, the specific features that justify it.
10. **Limitations and honesty:** what is mocked, what is custodial, what is not done.
11. **Run locally / env:** prerequisites, `.env.example` explanation, offline mode (`LEDGER=memory`).
12. **License and credits.**

### 2.2 Style rules
- Use "do X → see Y" phrasing for every verification step.
- Include exact commands and exact expected output snippets (short).
- Link the demo video and demo post.
- Do not claim features that are not implemented (especially `security`-track function-call keys, NEAR Intents, or real social login if only `GuestAuth` shipped).
- Keep it under ~250 lines; link out to `/docs` for depth.

## 3. What the human must decide (never guess, never fill placeholders)

Collect **all** of these from the human:

| Field | Rule |
| --- | --- |
| Project name | 2–80 chars, unique across the hackathon (case-insensitive) |
| Team lead's name | 2–80 chars |
| Contact email | Valid, max 160 chars, unique |
| Team size | Exactly one of: `1`, `2 - 4`, `4 - 6`, `6 - 8`, `8+` |
| Tracks | One or more of the ids below, each once, only tracks the project truly fits |
| NEAR account for prize payout (mainnet) | Named account like `team.near` or a 64-char implicit account; **the team creates it; never invent or create one** |
| Public repository URL | http(s), max 300 chars |
| Public demo post URL (X) | http(s), max 300 chars; the human writes and publishes it |
| What they built | 20–2000 chars: the problem, the product, how NEAR fits in |
| Consent | Read verbatim and get a yes: *"I agree that the organizers may use these details to review our submission and contact us about this hackathon."* |

**Track ids:** `defi`, `thesis`, `privacy`, `business`, `swarms`, `security`, `agents`, `realworld`, `games`, `nfts`, `memecoins`, `social`, `open`.
**Our expected set:** `games` (primary); `agents` and/or `swarms` if implemented as described; `security` only with demonstrable function-call-key limits.

## 4. Draft "what we built" text (edit with the human; 20–2000 chars)

> Saltmarket is a harbor-town trading game where the economy is the gameplay. Players run a stall, buy fish, salt, and timber, cure fish, and sell to a visiting ship captain and to NPC merchants. Every NPC has its own wallet, a budget, a decision rule, and a plain-English reasoning line for each price change. Players can buy an Exchange Hall and earn fees from trades; rival halls compete on fees and NPCs migrate to the cheaper venue. Sinks, faucets, and feedback loops let the economy adapt without developer patches; a Storm event cuts fish supply and the town visibly absorbs the shock. On NEAR, goods and currency are a NEP-245 multi-token contract, each NPC has a sub-account, and balances are settled in batches while the server relays gas, so players sign in with a social login and never see a wallet.

*(Adjust to match what is actually implemented. Remove any claim that isn't true at submission time.)*

## 5. Demo post (the human publishes; you draft)

Requirements: public post on X; says the project was **built at NEAR HACKATHON**; has the demo video or screenshots attached.

**Draft template:**
> Built at NEAR HACKATHON: Saltmarket 🐟🧂 A harbor town where the economy is the game. NPC merchants have their own wallets and explain every price move. Hit the Storm button and watch the whole town adapt, no dev patch. Players never see the chain. [video] [repo link]

Keep it short, factual, and consistent with the README. The human posts it and gives you the URL.

## 6. Pre-submission checklist

**Repository**
- [ ] No secrets in files or git history (run a secret scan; check `.env*`, `*.key`, credentials folders).
- [ ] `.env.example` has names only.
- [ ] `LICENSE` added (confirm choice with the human).
- [ ] README complete per §2; commands verified on a **clean clone**.
- [ ] Docs in `/docs` are current; `CLAUDE.md` Status updated.
- [ ] Tests green; Storm scenario passes.
- [ ] Repository made **public** by the human (confirmed) before 2026-11-14T23:59:00Z.

**Chain**
- [ ] Contract deployed on **testnet**; id and explorer links in README.
- [ ] Sample settlement tx hashes included; reconciliation clean.
- [ ] Custody model and off-chain/onchain split stated honestly.

**Demo**
- [ ] Video recorded (see `TESTING_AND_DEMO.md` §5) and uploaded where the human chooses.
- [ ] X post published by the human; URL collected.

**Submission**
- [ ] Re-read https://nearhackathon.com/HACKATHON.md for changes.
- [ ] All fields collected from the human (no guesses).
- [ ] `requestId` generated once and saved locally (not committed).
- [ ] Full JSON payload printed to the human; explicit "yes" received.
- [ ] Submitted **once**.
- [ ] 201 response noted; human informed.

## 7. Payload template (placeholders only; never send these values)

Generate `requestId` once: `uuidgen | tr A-Z a-z` or `node -e 'console.log(crypto.randomUUID())'`. Reuse the same value on retries after network errors or 503s.

```json
{
  "requestId": "<generated-uuid-lowercase>",
  "projectName": "<from human>",
  "leadName": "<from human>",
  "email": "<from human>",
  "size": "<one of: 1 | 2 - 4 | 4 - 6 | 6 - 8 | 8+>",
  "tracks": ["games"],
  "nearAddress": "<human's mainnet payout account, lowercase>",
  "projectUrl": "<public repo URL>",
  "demoUrl": "<public X post URL>",
  "idea": "<20-2000 chars, from §4 after human edits>",
  "consent": true
}
```

- Send exactly these fields. **Do not add others.** (`website` is a spam trap; any non-empty value rejects the submission.)
- Body must be ≤ 12,000 bytes.
- `nearAddress` must match `^(([a-z\d]+[-_])*[a-z\d]+\.)*([a-z\d]+[-_])*[a-z\d]+$`, 2–64 chars, lowercase.
- Do not commit the payload file; it contains personal data.

```sh
curl -sS -X POST https://nearhackathon.com/api/applications \
  -H 'Content-Type: application/json' \
  -d @payload.json   # payload.json is local only and listed in .gitignore
```

## 8. Response handling

| Status | Meaning | Action |
| --- | --- | --- |
| 201 | Saved | Done; tell the human |
| 400 | Field validation failed | Fix fields listed in `fields`; resend with the **same** `requestId` |
| 403 | Not open, or looked cross-site | Tell the human; **do not retry** |
| 409 | Project name or email already used | The team already submitted; ask the human; do not rename to get around it |
| 413 | Body too large | Shorten the description |
| 503 | Storage temporarily unavailable | Wait a minute; resend with the same `requestId` |

If the human prefers a form over the API, the same fields are at https://nearhackathon.com/#apply.

## 9. Final-day timeline (UTC)

| When | What |
| --- | --- |
| Nov 13 | Public-repo prep, secret scan, README final, video recorded |
| Nov 14, by ~06:00 | Clean-clone verification; repo public (human confirms) |
| Nov 14, by ~09:00 | Human publishes X post; gives URL |
| Nov 14, by ~12:00 | Payload confirmed and submitted (target) |
| Nov 14, 23:59 | **Hard deadline.** Nothing may be pending |
