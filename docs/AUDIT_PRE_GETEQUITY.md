# PawaSave — Pre-getEquity Integration Audit

**Date:** 2026-09-14
**Branch audited:** `audit-v2-remediation-and-flint-onramp` @ `d4b42e0`
**Scope:** full repo — Solidity contracts, Supabase schema/RLS/RPCs, Next.js API routes, custody & ramp layer, the existing equity/getEquity code, CI and dependency posture.
**Purpose:** establish a known-good baseline before building out the getEquity integration.

---

## 1. Method — what was actually executed, not just read

| Check | Command | Result |
|---|---|---|
| Contract compile | `npx hardhat compile` | ✅ 29 files, solc 0.8.20 |
| Contract tests | `npx hardhat test` | ✅ **85 passing**, 0 failing |
| Frontend typecheck | `npx tsc --noEmit` | ✅ clean, exit 0 |
| Dependency audit | `npm audit --audit-level=high` | ❌ **exit 1** — 1 critical, 5 high |
| Frontend tests | — | ⚠️ **none exist** (0 test/spec files) |

**CI is currently red.** `.github/workflows/ci.yml` runs `npm audit --audit-level=high` as a non-optional step in the `frontend` job. It exits 1 today, so every push to `main` and every PR fails regardless of code quality. Fix this before opening getEquity PRs or the signal is worthless.

Critical/high advisories: `next` (**critical** — Image Optimizer DoS via `remotePatterns`; installed 14.2.35), `nodemailer` (`resolveContent()` bypasses `disableFileAccess`/`disableUrlAccess`), `ws`, `postcss`, `nanoid`, `browserslist`. The `ws`/`viem` chain comes in via `@hyperbridge/sdk` — which is the HyperFX dependency the live equity path relies on, so it can't simply be dropped.

---

## 2. Headline verdict

The codebase is **more mature than its documentation suggests and less safe than its recent commit messages imply.**

Three things you should internalise before we touch getEquity:

1. **getEquity is already half-built, and the half that exists cannot run.** `lib/getequity.ts` calls two Postgres RPCs — `place_getequity_order` and `settle_getequity_order` — that **do not exist in any migration**. I verified this: they appear only inside the `pg_proc`-driven revoke loops of migrations 077/078, which silently skip missing functions. Flipping `GETEQUITY_ENABLED=true` today returns `400 "Could not place order"` on every buy. The integration is not "nearly done"; its persistence layer was never written.

2. **There is a live, unbacked double-credit bug latent in the naira on-ramp**, armed by a config change rather than a code change (PS-C-01 below). It must be fixed before anything else, because it is the kind of bug that silently mints liabilities.

3. **There is no double-entry ledger.** Balances are mutable columns (`wallets.usdc_balance_micro`), `transactions` is a single-entry journal with no counter-account, and — critically — **that journal is still client-writable**. Every "the numbers reconcile" claim in this system currently rests on trust.

The recent `security(...)` commits are genuine, well-reasoned work (migrations 077–082 are the best code in the repo). But they fixed the tables a specific staging harness probed, and stopped there. The same bug class is still live on the esusu/ajo tables.

---

## 3. State of the getEquity integration

There are **two unrelated investment integrations** sharing DB tables, a UI tab and the custody wallet:

| | `lib/equity-broker.ts` — **LIVE** | `lib/getequity.ts` — **DARK / non-functional** |
|---|---|---|
| Asset | Coinbase "B20" tokenised US stocks on Base (8 dp) | GetEquity Nigerian RWA (T-bills, funds, REITs, pre-IPO) |
| Rail | HyperFX (cNGN↔USDC) + Aerodrome/Uniswap V3 | GetEquity `Market` contract directly |
| Integration style | DEX router calls | **Contract layer, not their REST API** — deliberate, to keep cNGN settlement and avoid Flutterwave fiat fees (`getequity.ts:14-18`) |
| Network | Base mainnet | Base **Sepolia** (84532); mainnet "pending ~Aug 2026" |
| Maturity | Production-hardened: two-leg atomicity, `settling` state, reconcilers, attempt caps, divergence ledger, cost-basis math | Skeleton — no DB layer, no sell path, no accounting |

Neither is a fallback for the other. `getequity.ts` has three live callers, so it is not dead code — it is *reachable* dead weight.

### Blockers to getEquity going live

| ID | Issue | Location |
|---|---|---|
| **GE-1** | `place_getequity_order` / `settle_getequity_order` don't exist. Verified absent from all 71 migrations. | `api/invest/getequity/route.ts:147,167,177` |
| **GE-2** | `asset_type = 'rwa'` is **unrepresentable**. `portfolio_holdings.asset_type` CHECK is `IN ('tokenized_stock','pre_ipo')` and no migration widens it. Yet `refreshRwaPrices` filters `.eq('asset_type','rwa')` and the UI renders it. Three readers, zero possible writers. | `032:21` vs `getequity.ts:334` |
| **GE-3** | **Decimal scaling is wrong and self-flagged.** `buyWithCngn` hardcodes `ONE = 10n ** 18n` while `listAssets` correctly reads per-asset `decimals()` and `refreshRwaPrices` correctly uses `10n ** BigInt(a.decimals)`. It also compares a 6-dp cNGN budget against `totalCost` denominated in the asset's own `payoutToken()` base units. Any non-18-dp RWA is mispriced by orders of magnitude. The code says so twice. | `getequity.ts:262`, `236-240`, `318-324` |
| **GE-4** | **Client-supplied token address is never allowlisted.** The POST body's `token` goes straight into `buyWithCngn`, which calls `payoutToken()` on it and then `approve(MARKET, MAX_UINT256)` on whatever that returns. `isAssetTradeable`/`getRegisteredTokens` are never consulted on the write path. | `getequity/route.ts:126` → `getequity.ts:270-275` |
| **GE-5** | `claimPayout` is a stub returning `claimed: 0n`. The yield cron moves customer float into NTBL/ARMNGF and **writes to no table at all** — only `console.info`. Deployed principal and earned yield are entirely unaccounted. | `getequity.ts:302-311`, `cron/getequity-yield` |
| **GE-6** | No RWA sell/redeem path. `buyAsset`/`sellAsset` are exported with zero callers; UI hardcodes `sellable = asset_type === 'tokenized_stock'`. | `invest-view.tsx` |
| **GE-7** | The POST is fully synchronous with **no `maxDuration`** override, so a slow on-chain buy can be killed by the proxy *after* the debit — with no reconciler for getEquity orders at all. | `getequity/route.ts` |
| **GE-8** | `cron/getequity-yield` is **not in `ops/cron/crontab`** (verified — only the two equity reconcilers are). No `EQUITY_*`/`GETEQUITY_*` entries in either `.env.example`. | `ops/cron/crontab` |

### Inherited weaknesses the live equity path will pass on to getEquity

- **No reconciler for `pending` orders — in either integration.** Both money routes debit, then return early and settle in a fire-and-forget `void (async () => …)()`. On process restart or redeploy, the order stays `pending` forever with the cNGN debited. `equity-buy-reconcile` only scans `status='settling'`. There is no `fail_stale_equity_orders` analogue to the one that exists for withdrawals (`035`). **This is the largest unaddressed money-loss path on the live stock flow.**
- **Order placement is not idempotent.** No client idempotency key, no dedupe window, no "one open order per user/symbol" constraint. Two rapid POSTs create two orders and two debits; the only guard is a client-side `busy` flag. (Balances are safe — `place_equity_order` uses `SELECT … FOR UPDATE` — but duplicate orders are not.)
- **Buys have no fair-value floor.** `minOut` bounds quote→execution drift only, not price impact already inside the quote. A buy into a thin pool fills at any price; the only assertion is `shares > 0`.
- **Sell share-count mismatch:** `tokenBase` is clamped down to actual custody holdings, but `settle_equity_sell` uses the *requested* `s.shares`. When the clamp bites, the user is debited more shares than were sold and the difference is silently absorbed.
- **`1600` hardcoded as the NGN/USD fallback in three separate places** (`equity-prices.ts:34`, `invest/equity/route.ts:71-76`, `invest/equity/sell/route.ts:104`). Divergence is a live risk.
- **Yahoo Finance is the only price source** — unauthenticated, spoofed `User-Agent`, no key, no fallback. When it fails, the sell fair-value floor is skipped entirely.

---

## 4. Critical findings

### PS-C-01 — Every Strails naira deposit double-credits the moment webhook verification starts working
**Verified directly.** The two crediting paths pass **different idempotency keys for the same deposit** into the same function:

- `api/strails-webhook/route.ts:175` → `reference = depositId` (raw)
- `api/cron/strails-reconcile/route.ts:68` → `reference = \`strails_${ref}\`` (prefixed)

`credit_strails_deposit` de-dupes on `EXISTS (SELECT 1 FROM transactions WHERE reference = p_reference)` (`067:46-48`). Two different strings → **both credit**. The webhook's own pre-check (`:153`) also queries the unprefixed key, so it cannot see the cron's row either, and `transactions_reference_uniq` (migration 081) doesn't help because the strings differ.

This is masked *only* because `verifyStrailsWebhook` currently never succeeds. Setting `STRAILS_WEBHOOK_SECRET` correctly — the stated intent of the code comments — instantly doubles every naira deposit against an unbacked ledger.
**Fix:** one canonical key (`strails_${ref}`) in both paths, before anyone touches Strails config.

### PS-C-02 — `esusu_groups` `FOR ALL` RLS policy → arbitrary self-credit
`001_initial.sql:78`:
```sql
create policy "Owner manages group" on public.esusu_groups for all using (owner_id = auth.uid());
```
`FOR ALL` + `USING` + no column restriction, on a table holding `pot_balance_kobo`. A group owner, from the browser with the anon key and a login, can `UPDATE esusu_groups SET pot_balance_kobo = <anything>, status='active'`. Then `process_esusu_payout` — **granted to `authenticated`** (`070:163`) — reads `v_payout_kobo := v_group.pot_balance_kobo` (`070:127`) and credits it straight to a wallet (`070:139`).

This is *exactly* the bug class migration 080 was written to close, on a table 080 never audited. 081's non-negativity constraints don't cover `esusu_groups`.

### PS-C-03 — `esusu_contributions` client INSERT forges "everyone paid"
`004_fees_locks_admin.sql:37` lets a member insert contribution rows with arbitrary `amount_kobo` and `cycle_number`. `process_esusu_payout` gates payout on `COUNT(DISTINCT ec.member_id) >= v_member_count` (`070:110-115`) — forged rows satisfy it without anyone paying. Compounded by there being **no unique constraint on `(group_id, member_id, cycle_number)`**.

### PS-C-04 — Single-point-of-compromise custody; AWS Secrets Manager is advisory
One BIP-39 mnemonic derives every user deposit address; one EOA holds the entire float. `getSecret()` **unconditionally falls back to `process.env`** (`secrets.ts:59`), and on any AWS error `loadBundle` returns the last-good cache then env. So:
- No KMS envelope encryption, no HSM, no MPC. Both the mnemonic and `CUSTODY_PRIVATE_KEY` sit in plaintext in the Node heap of every invocation.
- **Rotation can silently regress to a stale env key** on one AWS hiccup.
- **The gas funder defaults to the custody key** (`deposit-sweep.ts:80`), maximising exposure and nonce churn of the key holding all customer funds.
- `custodyAddress()` derives the private key just to read an address (`custody.ts:52-54`), pulling key material into read-only routes.

### PS-C-05 — ~145 of ~150 `SECURITY DEFINER` functions have unpinned `search_path`
`set search_path` appears only in migrations 073–082. Everything from `001`–`072` — `save_to_vault`, `withdraw_vault_atomic`, `credit_wallet`, `withdraw_lock`, `process_esusu_payout`, `settle_equity_order` — is definer-rights with a mutable `search_path`. Textbook privilege-escalation vector. `supabase/tests/search-path-resolution-check.sql` exists; the pinning was never applied retroactively.

### PS-C-06 — `api/xend-webhook` can double-credit and can credit a *successful* withdrawal
`:63-76` does a plain select-then-update with **no `.eq('status','pending')` guard on the update** — concurrent retries both credit. `:78-90` credits with **no `tx.type === 'deposit'` check** (contrast `api/webhook:110`), so a `completed`-withdrawal callback credits the user on top of a delivered payout. Currently gated off by `XEND_ENABLED` and blunted because `runXend` never populates the lookup column — but it is one env flag from live.

### PS-C-07 — Xend refunds apply an FX rate to a cNGN-denominated debit
Debit is naira × 1e6; refunds compute `amount / rate` (`ramp/route.ts:648-651,705`; `xend-webhook:131`). At rate ≈1400, a **₦100,000 failed withdrawal refunds ₦71**. Env-gated off, one flag from live.

---

## 5. High findings

| ID | Finding | Location |
|---|---|---|
| **PS-H-01** | **Off-ramp marked `completed` on *broadcast*; reconciler only scans `pending`.** `sendCngn` deliberately doesn't `.wait()`. A dropped/replaced cNGN transfer = user debited, bank unpaid, row permanently `completed`, and **nothing will ever detect it**. The reconciler already knows how to verify a hash — it just never sees these rows. Needs a `settling`/`broadcast` status. | `custody.ts:78-84`, `ramp/route.ts:906`, `cron/reconcile-withdrawals:120` |
| **PS-H-02** | **`admin/reconcile-withdrawals?autoCompleteOnchain` force-completes un-sent withdrawals.** It filters `.like('description','%on-chain:%')`, which also matches the **pre-send** marker `on-chain: settling → 0x…`. One admin click permanently completes withdrawals where no cNGN ever left custody, removing them from the reconciler's scope. Separately, `markFailed` flips status **without refunding**, diverging from every other failure path. Neither filters on current status. | `admin/reconcile-withdrawals:56-72` vs `ramp/route.ts:365` |
| **PS-H-03** | `transactions` is **still client-writable** (`001:57` "Users insert own txs"), and `enforceWithdrawalKycCap` sums that very table to enforce KYC limits. Acknowledged in 080/081 headers; still live. | `001_initial.sql:57` |
| **PS-H-04** | **Migrations 046–061 are missing from version control** (verified: 001–045 then 062–082). `pin_lock_status`, `record_pin_attempt`, `welcome_sent`, `kyc_tier` have no `CREATE` anywhere. Because `lib/pin-lockout.ts:37` **fails open** on RPC error, a DB rebuilt from git silently loses all PIN brute-force protection on a 4-digit secret — including on the withdrawal path. | `supabase/migrations/` gap |
| **PS-H-05** | **`/api/ussd` identifies users by a client-supplied `phoneNumber` under the service role**, with an *optional*, non-constant-time gateway secret accepted in the query string. Reaches PII, balances, PIN set/change and Ajo contributions, with no PIN lockout on that path. | `api/ussd/route.ts:46-59,115-120` |
| **PS-H-06** | **Admin auth is a single shared `ADMIN_PASSWORD`** — no MFA, no per-operator identity, no dual approval, no audit of who moved money. A body-password fallback **bypasses migration 031's 5-strike lockout** on every admin route except `/verify`, and `ADMIN_SESSION_SECRET` falls back to using `ADMIN_PASSWORD` as the HMAC signing key. Meanwhile `admin/revenue-withdraw` sends custody cNGN to a **body-supplied bank account with no ceiling and no destination verification**. | `admin-session.ts:11,17,62-68`; `admin/revenue-withdraw:96-127` |
| **PS-H-07** | **`strails-webhook` is an unauthenticated remote trigger for custody-signing work.** On verification failure it returns **HTTP 200** and calls `/api/cron/strails-reconcile` **supplying `CRON_SECRET` itself** — which is always, today. Any internet caller can drive the Strails sweep + `supplyToLend` loop: custody signing, gas spend, lease contention with real off-ramps, API quota burn. | `strails-webhook:47-75` |
| **PS-H-08** | **`transactions.reference` uniqueness collides with `credit_crypto_deposit`'s key.** 081 adds a unique index on `reference`; `credit_crypto_deposit` inserts `reference = p_tx_hash` while its own idempotency key is `tx_hash:log_index`. One Base tx paying **two** user deposit addresses credits the first and raises a unique violation on the second — on the initial scan and every rescan forever. Funds sit on-chain, uncredited, failure invisible. | `081:41-43` vs `026:105-113` |
| **PS-H-09** | **`depositWalletConfigured()` guard missing `await`** — `if (!depositWalletConfigured())` on a Promise is always false. **Verified**: correct at `deposit-sweep.ts:66` and `wallet/deposit-address:34`, broken at `deposit-scan.ts:174`. The cron's clean-skip is defeated and it 500s instead. | `deposit-scan.ts:174` |
| **PS-H-10** | **`deposit-sweep` signs with the custody key outside the custody lease.** `cron/sweep-deposits:57` calls `sweepDeposits()` unleased; it then builds a wallet from the custody key and sends gas top-ups concurrently with off-ramp `sendCngn` from other instances. Exactly the duplicate-nonce scenario the lease was built to prevent — and per PS-H-01 a replaced off-ramp tx is unrecoverable. | `deposit-sweep.ts:95,126` |
| **PS-H-11** | **Public RPCs are in the *write* path.** `writeRpcUrls()` always appends `mainnet.base.org`/`publicnode`/`llamarpc`, so an exhausted paid key silently moves **transaction signing** onto inconsistent public load balancers. No opt-out. | `rpc-provider.ts:66-76` |
| **PS-H-12** | **Four tables have RLS never enabled and DML never revoked** — `proxy_transfers`, `fixed_savings_rates`, `deposit_scan_state`, `revenue_journal`. 082 revokes only `TRUNCATE/REFERENCES/TRIGGER` and its header says *"DML is deliberately left alone."* Any authenticated user can rewrite `fixed_savings_rates` (the APY tier table) or **rewind `deposit_scan_state`** (the scanner cursor → rescan/re-credit deposits). | `018:6`, `022:8`, `026:63`, `023:18` |
| **PS-H-13** | **078 re-grants `withdraw_lock` and `break_savings_goal` to `authenticated`**, contradicting the design they were moved behind a server route for. `p_early` is caller-supplied; the non-early branch pays principal + projected interest. A signed-in user calling the RPC directly bypasses forfeiture enforcement. | `078:83`, `071:86` vs `use-data.ts:334-345` |
| **PS-H-14** | `esusu_members` self-insert with **attacker-chosen `payout_position`**; the payout recipient is selected `ORDER BY payout_position LIMIT 1`. Insert yourself with `-1` → first in the queue. | `001:94`, `070:114-117` |
| **PS-H-15** | **Revenue counter lost-update + decrement after unconfirmed broadcast.** `admin/revenue-withdraw` reads `platform_revenue_kobo`, sends cNGN (broadcast-only), then writes back `revenue - requested`, overwriting any concurrently-booked fee. If the post-send update fails, the cNGN is gone but the counter still shows it withdrawable → repeatable drain. | `admin/revenue-withdraw:75-145` |

---

## 6. Medium findings (abridged)

- **`ngnToCngnMicro` applies a fetched float FX rate to a 1:1-pegged asset** on the deposit-credit path. If `api.cngn.co` returns `0.998`, every fiat deposit is credited 0.2% short, `Math.floor` always truncates against the user, and the bad rate is cached with **no TTL**. The exact helper (`koboToCngnMicro`) already exists. — `ramp-rate.ts:150-153`
- **Money is floats-of-naira until the last moment.** No Decimal/BigInt money type. The off-ramp gross-up derives the same figure **four different ways** with `Math.round`/`Math.floor` mixed; they agree only because inputs happen to be integers. Inline refunds and reconciled refunds for the same row can differ by a kobo. — `ramp/route.ts:735-739,753,277,829`
- **FX rounding direction is wrong**: `offSendNaira = Math.round(amount / rate)` should be `Math.ceil` — the stated goal is delivering the *full* promised amount.
- **`maybeDebitForWithdrawal` is a non-atomic two-RPC sequence** — a successful `withdraw_cngn_pool` followed by a failing `debit_wallet` moves the shortfall out of the pool and never restores it.
- **All refunds credit spendable regardless of debit source**, silently converting pooled savings into spendable balance and drifting the 90%-pool invariant.
- **`custodyCngnTransferTo` matches on destination address only** — no amount check, no time window, `fromBlock: '0x0'`. An old transfer to a reused Flipeet address completes a new withdrawal.
- **`flipeet-webhook` auth is a static bearer token in the URL query string** with no replay protection — logged by every proxy and by Flipeet. Leaking it enables forged `failed` events that mint refunds. Both it and `strails-webhook` **credit amounts taken from the request body**, not from the persisted `amount_kobo`.
- **`strails-webhook` idempotency key degrades to `strails_${Date.now()}`** when no deposit id is present → a replay credits repeatedly. Its fee is also computed from a different base than the reconciler's.
- **Deposit scanner has no from-address filter** — any inbound cNGN to a deposit address is credited, so an operator refunding via a user's deposit address double-credits them.
- **No sweep ledger at all** — `deposit-sweep.ts` persists nothing; a sweep mining after the function timeout is invisible.
- **`verifyPayoutDestination` fails open** on provider 5xx, while the code elsewhere notes the Flipeet callback "often never fires."
- **Views bypass RLS** — none of ~13 views set `security_invoker`; `offramp_audit` and `yield_summary` are cross-user.
- **Non-constant-time secret comparisons**: `api/webhook:21` (`signature === hash`), `cron-auth.ts:19`. `verifyStrailsWebhook` also brute-forces a *matrix* of candidate secrets including `STRAILS_API_KEY` — signing-key confusion.
- **Schema drift**: `push_subscriptions` and `ajo_invite_codes` are queried by code but have **no `CREATE TABLE` anywhere**. Duplicate migration ordinals (three `007`s, two `017`s) mean which function definition wins depends on filename sort order. `break_savings_goal` and `complete_savings_goal` have **two coexisting overloads** because the one-arg versions were never dropped — PostgREST resolution depends on the argument names the caller sends.
- **README is badly stale** — it documents a NestJS/Prisma/BullMQ backend with `/api/auth/register` endpoints that does not exist. Reality is Next.js App Router + Supabase. `ops/env-checklist.md` instructs running only migrations 032 + 062, omitting 063–076, several of which fixed money bugs.

---

## 7. Smart contracts

**Deployed on Base mainnet** (`deployments/`): `PawasaveLend` `0x5583…B7cF`, `PawasaveLendStrategy` `0x5C8b…7373`, `PawasaveAutoVault` `0xcBA4…c37b`, `PawasaveCreditLine` `0x5056…1B13`, `PriceOracle` `0x6385…183c`, `InterestRateModel` `0x1881…e8E7`. Owner `0x04A6…9109`, treasury/feeRecipient `0xf1d5…cc22`, oracleKeeper `0x22b1…9719`. One credit-line deployment (`0x723C…3F06`) is marked abandoned after a V2-HIGH-01 principal/interest fix.

**This is the strongest part of the codebase.** The contracts show real audit-remediation discipline, with findings traceable in comments (FIND-SC-01…20, V2-*):

- `PawasaveAutoVault` is ERC4626 with `_decimalsOffset() = 6` (inflation resistance), **donation-proof `totalAssets()`** via an internal `deployedAssets` counter, O(1) lock accounting, `depositFixed` requiring `receiver == msg.sender` (anti-grief), a **48h timelock on strategy changes** with interface sanity checks, `Ownable2Step`, `ReentrancyGuard`, `Pausable`, a real `emergencyWithdraw`, and `_withdraw` acting on the amount a strategy *actually* returns with a `require(balanceOf >= assets)` backstop.
- `PriceOracle` has a **25% deviation circuit breaker**, a per-token `minPrice` floor that applies even to `forceSetPrice` (closing the first-set near-zero-price hole), `MAX_PRICE_AGE = 1 hour`, and a keeper separated from owner.
- `PawasaveLend` has supply caps, per-user caps, tenor bounds, reserve/insurance split, and pausability.

**Contract-layer concerns:**

1. **Ownership is a plain EOA, not a multisig or timelock.** `0x04A6…9109` owns the vault, pool, oracle and credit line. `Ownable2Step` protects against a mistaken transfer but not against key compromise. `collectReserves`, `setTreasury`, `updateFeeRecipient`, `setCollateralFactor` and `emergencyWithdraw` are all one key away. A 2-of-3 Safe here is the single highest-leverage security upgrade available.
2. **Keeper centralisation.** Oracle freshness, liquidation, harvest, APY updates and yield accrual all depend on crons hitting authenticated endpoints. Stale prices past `MAX_PRICE_AGE` block borrowing (fail-safe, good), but a stalled `liquidate` cron means insolvent loans simply aren't liquidated — there's no permissionless liquidator incentive to pick up the slack.
3. **Cron auth is `!==` string comparison** (`cron-auth.ts:19`) — not timing-safe, though it correctly **fails closed** when `CRON_SECRET` is unset.
4. **Test coverage is good on remediated findings, thin on adversarial scenarios.** 85 tests pass and each maps to a specific audit finding. Missing: fuzz/invariant tests, explicit reentrancy tests, multi-user share-price-manipulation sequences, and liquidation-cascade / bad-debt scenarios. Tests are wired into CI and genuinely run — but the `contracts` job shares the failing `npm audit` step.
5. `rebalance()` silently no-ops when conditions aren't met yet still isn't gated on `lastRebalanceTime` in the early-return path.

---

## 8. What's genuinely good — preserve these

Worth naming explicitly so we don't regress them while working on getEquity:

- **The custody lease** (`custody-lease.ts` + migration 073) is a properly-built Postgres-backed mutex over the single custody EOA: jittered backoff, TTL refresh at TTL/3, `AsyncLocalStorage`, reentrancy via `NestedLease`, `assertHeld()` before irreversible actions, and it **fails closed**.
- **Two-leg atomicity on the equity path.** `EquityBuyStockPending`/`EquitySellCngnPending` exceptions exist specifically so a failure *after* cNGN has left custody does **not** refund the customer out of company float. Migration 075's header documents learning this the expensive way on the sell side. The `settling` state, attempt caps, `equity_orders_needing_attention` view and `custody_divergences` ledger are all correct instincts.
- **Off-ramp ordering is right**: debit before any external call, no provider fallback for `off` (prevents a second debit), an explicit `alreadyRefunded` flag, `SELECT … FOR UPDATE` in `debit_wallet`, and a durable settlement marker written and read back before signing.
- **`FailoverWriteProvider`** correctly refuses to retry on `nonce|revert|insufficient funds|already known|replacement|underpriced` — the chain answered. Well-reasoned.
- **Migrations 077–082** are the best code in the repo: 082 uses an *event trigger* rather than `ALTER DEFAULT PRIVILEGES` (with the reasoning for why the latter fails) and asserts exactly one anon-callable function remains. 079 correctly diagnosed that `record_lock_forfeiture` compared a `uuid` to a `bigint`, so **the early-exit penalty had never once applied** — and dropped the bad overload rather than shadowing it.
- **`supabase/tests/adversarial-rls-writes.sql`** — an actual adversarial harness. Extend it rather than replacing it.

---

## 9. Recommended sequencing

**Phase 0 — unblock the pipeline (hours)**
1. Fix `npm audit` so CI is green: bump `next` (critical), `nodemailer`, `postcss`, `nanoid`; decide on the `@hyperbridge/sdk` → `ws`/`viem` chain (pin, override, or accept with a documented exception rather than leaving CI red).
2. Fix **PS-C-01** (Strails key mismatch) and **PS-H-09** (missing `await`). Both are one-line, both are latent-armed.

**Phase 1 — close the fund-loss holes (days)**
3. **PS-H-01**: add a `settling`/`broadcast` status to withdrawals and let the reconciler verify those hashes.
4. **PS-H-02**: fix the `%on-chain:%` matcher and make `markFailed` refund.
5. **PS-C-02/03/PS-H-14**: drop the esusu `FOR ALL` policy, restrict contribution/member inserts, add the `(group_id, member_id, cycle_number)` unique constraint. Extend the adversarial harness to cover esusu before declaring it fixed.
6. **PS-H-12/PS-H-13**: enable RLS on the four tables, revoke DML, re-revoke `withdraw_lock`/`break_savings_goal` from `authenticated`.
7. **Pending-order reconciler** for equity — needed by both integrations, and by getEquity on day one.

**Phase 2 — build getEquity properly**
8. Write the missing migration: `place_getequity_order`/`settle_getequity_order` with the same two-phase shape as 075 (`settling` state, attempt cap, no-refund-after-leg-1), and widen `asset_type` to include `'rwa'` on both `portfolio_holdings` and `equity_orders`.
9. Fix the decimal scaling (GE-3) against per-asset `decimals()` and `payoutToken()` decimals — exactly as `refreshRwaPrices` already does correctly. Add a scaling unit test.
10. Allowlist the client-supplied token against `getRegisteredTokens()` + `isAssetTradeable()` before signing (GE-4).
11. Make `claimPayout` return the real claimed amount and persist deployed principal + claimed yield (GE-5). Do **not** schedule `getequity-yield` until there's a ledger behind it.
12. Add `maxDuration` and move the buy off the synchronous path (GE-7); document the env surface in `.env.example` (GE-8).

**Phase 3 — structural, in parallel**
13. Move contract ownership to a 2-of-3 Safe. Highest security-per-effort item in the repo.
14. Pin `search_path` on all pre-073 `SECURITY DEFINER` functions (PS-C-05).
15. Revoke client INSERT on `transactions` and move the six `use-data.ts` ledger writes server-side (PS-H-03) — this is the prerequisite for the ledger ever being trustworthy for reconciliation.
16. Split custody: separate gas-funder key, and get `CUSTODY_PRIVATE_KEY` behind real KMS rather than an env fallback (PS-C-04).
17. Rewrite the README. It describes a system that doesn't exist.

---

## 10. Two questions I can't answer from the code

1. **Is the getEquity mainnet deployment actually live yet?** The code says Base Sepolia with mainnet "pending ~Aug 2026" (that date has now passed). The whole scaling question in GE-3 needs a real sandbox quote to resolve — the code itself asks for exactly this verification twice. Do you have live GetEquity credentials/addresses?
2. **How much real money is currently in the vault/lend pool?** That determines whether Phase 1 and the Safe migration are urgent or merely important.
