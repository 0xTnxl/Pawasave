# PawaSave — Full Security & Code-Quality Audit (v3)

**Date:** 2026-09
**Auditor scope:** third-round (v3) source-level security & code-quality review
**Overall risk rating:** **MEDIUM–HIGH** (driven by centralization / hot-key custody and by source-vs-deployed drift, not by a confirmed exploitable flaw in the newly written code)

## 1. Scope statement

This is a fresh, third-round review of the PawaSave protocol spanning the money-handling
Solidity contracts (highest priority), the oracle/keeper scripts, operational secret and key
custody, the `strails-relay` service, the Supabase RPC/RLS grant model, and the Next.js
frontend/money-flow backend. It is a **source-level review**: dependencies are not installed
in the review environment, so no compilation, unit-test run, or on-chain interaction was
performed. Findings are grounded in the actual repository source. Items already marked
done/LIVE/won't-fix in `SECURITY_AUDIT_REMEDIATION.md` (v1, 40 findings) and
`AUDIT_V2_REMEDIATION.md` (v2, 43 findings) are **excluded** and are not re-reported as new;
where a prior item is re-raised it is explicitly tagged with the prior ID and a justification.

A critical framing point for everything below: the **7d contract stack is already LIVE on
Base mainnet** (addresses in `SECURITY_AUDIT_REMEDIATION.md` section 7d), while several v2
"fixed in source" changes are **not yet deployed**. See [Section 12 — Deployed vs. source drift](#12-deployed-vs-source-drift).

## 2. Executive summary

Headline findings:

- The protocol is **heavily owner/keeper-centralized by design**. A single owner key (the Gnosis
  Safe) can move fees, swap the IRM, `forceSetPrice` past the deviation breaker, draw
  uncollateralized `CreditLine` funds to any address, and withdraw idle `CreditLine` liquidity.
  A compromised owner key is the single largest risk to user funds. (V3-SC-01)
- The **price oracle is a single hot EOA keeper** feeding USDC/USDT prices from one primary
  source (Flint) with a public free-tier fallback, no on-chain Chainlink cross-check, and a
  1-hour staleness window. This is a manipulation / stale-liquidation surface. (V3-OK-01, V3-OK-02)
- **Overdue-but-healthy loans are liquidatable at a 10% bonus** purely for being past
  `dueDate + grace`. This is intended maturity enforcement, but it lets a fully-collateralized
  borrower lose 10% of seized collateral for lateness; it deserves an explicit policy decision. (V3-SC-02)
- **Hot keeper and custody keys** (oracle keeper, liquidation keeper, and the custody wallet that
  holds *all* user funds) sign from private keys resolved via `lib/secrets.ts`. The move to
  KMS/HSM signing is still open (re-raise of FIND-SC-21 / FIND-3P-05/06), and the deployer key is
  documented as **exposed** and still needs rotating out of the Safe signer set. (V3-OPS-01, V3-OPS-02)
- The Supabase grant model was **default-open**: migrations `077`/`078` retro-actively revoked
  `anon`/`PUBLIC EXECUTE` on money-moving `SECURITY DEFINER` RPCs that any unauthenticated caller
  could reach and abuse via a `p_user_id` parameter. This was patched out-of-band as hotfixes;
  the **root cause (no default-deny posture / no lint gate)** remains and must be closed. (V3-DB-01, V3-DB-02)
- Provider webhooks (Flipeet) are authenticated only by a **secret token embedded in the
  callback URL**; the provider does not sign callbacks. The `flipeet-webhook` also **auto-routes
  unmatched deposits** to a user via `get_user_for_proxy_member` using a provider-supplied
  beneficiary id, trusting provider input to credit an account. (V3-BE-01, V3-BE-02)
- **CI runs Slither with `fail-on: none`** (informational only) and has **no fuzz/invariant
  testing** for the lending math; contract tests are TypeScript/Hardhat unit tests only. For a
  fund-holding protocol this is under-gated. (V3-CI-01, V3-CI-02)
- `strails-relay` is an **over-broad authenticated proxy** of a key-bearing upstream: any caller
  with the shared secret can forward any path/method to Strails; there is no per-route allowlist. (V3-RL-01)

**Single most important recommendation:** Before scaling past the beta cohort, bundle the v2
"fixed-in-source" contract changes into a **v3 redeploy + re-audit**, and in the same window move
the oracle/liquidation/custody signing keys to **KMS/HSM** and rotate the exposed deployer key out
of the Safe signer set. The on-chain code that is *actually live* today lags the reviewed source.

## 3. Methodology & scope

- **Type:** manual source-level review of the repository at the current `main` commit. No build,
  test, or on-chain call was performed (dependencies not installed in the review environment).
- **Components reviewed:**
  - Smart contracts: `contracts/PawasaveAutoVault.sol`, `contracts/lending/PawasaveLend.sol`,
    `contracts/lending/PawasaveCreditLine.sol`, `contracts/lending/InterestRateModel.sol`,
    `contracts/lending/PriceOracle.sol`, `contracts/strategies/PawasaveLendStrategy.sol`,
    `contracts/IStrategy.sol`.
  - Oracle & keepers: `scripts/oracle-keeper.ts`, `frontend/src/app/api/cron/liquidate/route.ts`.
  - Operational secrets & key custody: `frontend/src/lib/secrets.ts`, `frontend/src/lib/custody.ts`,
    `generateKeys.js`, `sign-xend.js`, `.env.example`, `scripts/transfer-ownership.ts` (referenced).
  - `strails-relay/index.js`.
  - Supabase backend: `supabase/migrations/077_revoke_service_only_rpcs.sql`,
    `supabase/migrations/078_revoke_anon_execute.sql`.
  - Frontend & money-flow backend: `frontend/src/app/api/ramp/route.ts`,
    `frontend/src/app/api/flipeet-webhook/route.ts`, `frontend/src/app/api/admin/verify/route.ts`,
    `frontend/src/app/api/proxy/route.ts`, `frontend/src/lib/cron-auth.ts`.
  - CI/CD: `.github/workflows/ci.yml`.
- **Exclusions:** every item marked done/LIVE/won't-fix in the two prior trackers. Re-raised items
  are tagged with the prior ID.
- **Severity model:** Critical / High / Medium / Low / Informational, plus an explicit
  **Design/Centralization** label for inherent, by-design trust risks (these are mitigations-not-bugs).

## 4. Severity summary

| ID | Title | Component | Severity |
|----|-------|-----------|----------|
| V3-SC-01 | Owner powers over user funds (fees, IRM swap, forceSetPrice, CreditLine draw/withdraw) | Smart Contracts | Design/Centralization (High impact) |
| V3-SC-02 | Overdue-but-healthy loans liquidatable at 10% bonus | Smart Contracts | Medium |
| V3-SC-03 | CreditLine is fully uncollateralized + owner-can-draw-to-any-address | Smart Contracts | Design/Centralization (High impact) |
| V3-SC-04 | Borrow-balance / index rounding leaves un-repayable dust | Smart Contracts | Low |
| V3-SC-05 | Strategy `_sharesForAssets` round-up vs. bad-debt / falling exchangeRate | Smart Contracts | Low |
| V3-SC-06 | `InterestRateModel` per-second truncation + simple-interest reliance on frequent cron | Smart Contracts | Informational |
| V3-SC-07 | Lending pool uses single-step `Ownable` (vault uses `Ownable2Step`) | Smart Contracts | Low |
| V3-SC-08 | Liquidation `seizeAmount` can exceed intended when protocol-cut math meets dust | Smart Contracts | Low |
| V3-OK-01 | Oracle keeper is a single hot EOA, single primary rate source, no on-chain cross-check | Oracle & Keepers | High |
| V3-OK-02 | 1-hour staleness window enables stale-price liquidations / borrows | Oracle & Keepers | Medium |
| V3-OK-03 | Liquidation keeper approves `MaxUint256` cNGN and scans by event lookback | Oracle & Keepers | Low |
| V3-OPS-01 | Hot keeper/custody keys not yet on KMS/HSM | Operational Secrets & Key Custody | High |
| V3-OPS-02 | Exposed deployer key still in Safe signer context | Operational Secrets & Key Custody | High |
| V3-OPS-03 | RSA private key written to working directory by `generateKeys.js` | Operational Secrets & Key Custody | Medium |
| V3-OPS-04 | Secrets Manager failure silently falls back to `process.env` / stale cache | Operational Secrets & Key Custody | Low |
| V3-RL-01 | `strails-relay` is an over-broad authenticated proxy of a key-bearing upstream | strails-relay | Medium |
| V3-DB-01 | Default-open PostgREST grant model on SECURITY DEFINER RPCs (root cause) | Supabase Backend | High |
| V3-DB-02 | `auth.uid() IS NOT NULL AND != p_user_id` guard is a no-op for anon | Supabase Backend | High |
| V3-DB-03 | No default-deny / CI lint that new DEFINER funcs stay off PUBLIC/anon | Supabase Backend | Medium |
| V3-BE-01 | Webhook auth by secret-token-in-URL; provider does not sign callbacks | Frontend & Money-Flow Backend | Medium |
| V3-BE-02 | `flipeet-webhook` auto-routes unmatched deposits via provider-supplied id | Frontend & Money-Flow Backend | Medium |
| V3-BE-03 | Off-ramp debit→provider→refund path is multi-step and not atomic (TOCTOU window) | Frontend & Money-Flow Backend | Medium |
| V3-BE-04 | Admin auth is single password, no MFA | Frontend & Money-Flow Backend | Low (re-raise, accepted-by-decision) |
| V3-CI-01 | Slither runs `fail-on: none` (informational only) | CI/CD & Testing | Medium |
| V3-CI-02 | No fuzz/invariant testing for lending math; no frontend unit tests | CI/CD & Testing | Low |

Counts: Critical 0 · High 6 · Medium 8 · Low 7 · Informational 1 · Design/Centralization 2 (High impact).

## 5. Smart Contracts

### V3-SC-01 — Owner powers over user funds (Design/Centralization, High impact)
**Location:** `PawasaveAutoVault.sol` (`updatePlatformFee` up to 1500 bps, `updateFeeRecipient`,
`proposePrimaryStrategy`/`executePrimaryStrategy`, `emergencyWithdraw`, `setDepositCap`),
`PawasaveLend.sol` (`setReserveFactor` ≤30%, `setInterestRateModel`, `setCollateralFactor`,
`setOriginationFee`, `setGracePeriod`, `setMaxTenor`, `collectReserves`, `setTreasury`),
`PriceOracle.sol` (`forceSetPrice`, `setMaxDeviation`, `setKeeper`), `PawasaveCreditLine.sol`
(`draw`, `withdrawLiquidity`, `writeOff`, `setCreditLimit`).

**Description:** The owner (the Gnosis Safe per the trackers) holds broad economic control. Notably
`PriceOracle.forceSetPrice` bypasses the deviation circuit breaker entirely (bounded only by the
per-token `minPrice` floor *if set*), the vault fee can be raised to 15%, the IRM can be swapped
via `setInterestRateModel`, and `CreditLine` idle liquidity can be withdrawn by the owner.

**Impact:** A compromised owner key (or a malicious quorum of Safe signers) could distort collateral
prices to trigger unfair liquidations or enable bad-debt borrows, redirect fees, or drain idle
`CreditLine` liquidity. This is the single largest risk to user funds. It is a *centralization*
risk, not a code bug — the contracts behave as written.

**Recommendation:** (1) Move owner to a **timelock in front of the Safe** for parameter changes so
users get an exit window; (2) require a **per-token `minPrice` floor be set for every accepted
collateral** so `forceSetPrice` can never write a near-zero valuation; (3) publish the Safe signer
set and threshold and complete the 3-of-3 → 2-of-3 change with the exposed deployer removed; (4)
document every owner power and its blast radius in the protocol docs for user transparency.

### V3-SC-02 — Overdue-but-healthy loans liquidatable at 10% bonus (Medium)
**Location:** `PawasaveLend.sol` — `liquidate()` requires `!_isHealthy(borrower) || _isOverdue(borrower)`;
`_isOverdue` returns true once `block.timestamp > dueDate + gracePeriodSeconds` with debt > 0.

**Description:** A fully collateralized borrower who is merely past the loan maturity + 4-day grace
becomes liquidatable. The liquidator seizes collateral worth `repayAmount * (1 + liquidationBonusMantissa)`
(10% bonus), of which the protocol takes 2%. So a healthy but late borrower can lose ~10% of the
seized-collateral value for lateness, up to `closeFactor` (50%) of the debt per call.

**Impact:** Economic loss to on-time-collateralized-but-late borrowers; may be surprising and is a
UX/fairness and potential dispute risk in a consumer-lending product. It is by-design maturity
enforcement, but the 10% bonus on a *healthy* position is a deliberate policy choice that should be
made explicit.

**Recommendation:** Confirm this is intended. Consider a **reduced or zero liquidation bonus for the
"overdue-but-healthy" path** (a maturity-driven wind-down at par plus a small penalty) distinct from
the full 10% bonus reserved for genuinely under-collateralized positions, and surface the maturity /
grace terms prominently to borrowers.

### V3-SC-03 — CreditLine uncollateralized + owner-draw-to-any-address (Design/Centralization, High impact)
**Location:** `PawasaveCreditLine.sol` — `draw(address partner, uint256 amount, address to)` is
`require(msg.sender == partner || msg.sender == owner())` and sends cNGN to an arbitrary `to`.

**Description:** By design the credit line is uncollateralized (off-chain KYB/partner agreements are
the only backstop) and uses "managed custody" so the owner can draw on a partner's behalf to any
settlement address. This concentrates both credit risk and key risk.

**Impact:** A compromised owner key can `draw` up to each partner's `creditLimit` to any address and
`withdrawLiquidity` all idle funds. Credit risk itself is uncollateralized and unhedged on-chain.

**Recommendation:** This is inherent to the product. Mitigations: (1) keep `CreditLine` liquidity
minimal and topped-up just-in-time rather than pre-funded; (2) put owner draws behind a timelock or a
second approver; (3) enforce per-partner and global draw-rate limits on-chain; (4) ensure `CreditLine`
is included in the v3 re-audit before any deploy (it is currently undeployed per the trackers).

### V3-SC-04 — Borrow-balance / index rounding leaves un-repayable dust (Low)
**Location:** `PawasaveLend.sol` — `borrowBalanceCurrent` = `(principal * borrowIndex) / interestIndex`;
`_updateBorrowBalance` recomputes `current` the same way; `repay` clears `dueDate` only when
`principal == 0`.

**Description:** Integer division in the index math can leave 1-wei-scale principal dust after a
`type(uint256).max` "full" repay in edge cases (index growth between the `borrowBalanceCurrent` read
and the `_updateBorrowBalance` write is avoided by accruing first, but truncation in the multiply/
divide can still round). Dust principal keeps `dueDate` non-zero and the position technically open.

**Impact:** Low — a borrower could be left with a few micro-units of debt that keep the loan flagged
open / overdue. Not a fund-loss vector but a correctness/UX nuisance.

**Recommendation:** In `repay`, when the requested amount is `type(uint256).max` (full repay), zero the
`principal` and `dueDate` outright after transferring `currentDebt`, rather than relying on the
subtract-to-zero path; add a unit test asserting `borrowBalanceCurrent == 0` after a max repay across
several accrual points.

### V3-SC-05 — Strategy `_sharesForAssets` round-up vs. falling exchangeRate (Low)
**Location:** `PawasaveLendStrategy.sol` — `_sharesForAssets` rounds up (`(assets*WAD + rate - 1)/rate`);
`withdraw`/`harvest` clamp `shares` to `held`; `principal` is decremented by the actual `withdrawn`.

**Description:** If `PawasaveLend.exchangeRate()` falls (a bad-debt scenario where `totalPoolAssets`
drops), the strategy's `_currentValue` can dip below `principal`, so `harvest` correctly returns 0,
but the vault's `deployedAssets` accounting (updated on `withdraw`'s actual return) and the strategy's
`principal` can diverge from realizable value. The round-up on shares is safe for withdrawal
(redeems ≥ requested) but can slightly over-withdraw and shrink remaining principal faster than pure
value accounting.

**Impact:** Low — in normal (appreciating) conditions this is benign. In a bad-debt event the vault's
`totalAssets()` (idle + `deployedAssets`) can overstate realizable value until a withdrawal reconciles
it, briefly inflating share price.

**Recommendation:** Add invariant tests for the vault↔strategy pair under a *falling* `exchangeRate`
(inject bad debt in `PawasaveLend` in a test), asserting share price never lets an early redeemer
extract more than their pro-rata realizable share.

### V3-SC-06 — IRM per-second truncation + simple-interest cron reliance (Informational)
**Location:** `InterestRateModel.sol` constructor divides per-year rates by `SECONDS_PER_YEAR` (integer
truncation); `PawasaveLend.accrueInterest` uses linear `interestFactor = ratePerSecond * deltaTime`.

**Description:** Per-second rates truncate on construction, and interest is simple (linear) per accrual
window rather than compounded. Both are already documented (FIND-SC-24 informational won't-fix;
FIND-SC-15 documented). Re-raised only to note the **dependence on a frequently-run accrual cron**:
if the accrual cron stalls for a long gap, simple interest under-accrues vs. continuous compounding.

**Impact:** Informational — minor revenue under-accrual under cron outage; no fund loss.

**Recommendation:** Monitor/alert on accrual-cron liveness; keep inter-accrual gaps sub-daily as
designed. No code change required.

### V3-SC-07 — Lending pool uses single-step `Ownable` (Low)
**Location:** `PawasaveLend.sol`, `PawasaveCreditLine.sol`, `PriceOracle.sol`, `PawasaveLendStrategy.sol`
all extend OpenZeppelin `Ownable` (single-step). `PawasaveAutoVault.sol` uses `Ownable2Step`.

**Description:** The vault correctly uses two-step ownership transfer, but the lending pool, credit
line, oracle, and strategy use single-step `Ownable`. A `transferOwnership` typo to a wrong/dead
address is unrecoverable for those contracts.

**Impact:** Low but permanent-loss-of-control risk on an ownership transfer mistake (e.g. during the
Safe migration `scripts/transfer-ownership.ts` performs).

**Recommendation:** Standardize on `Ownable2Step` across all owned contracts in the v3 redeploy so
the Safe must `acceptOwnership()` on each, matching the vault. This also reduces the blast radius of
V2-INFRA-05 mistakes.

### V3-SC-08 — Liquidation seize-vs-protocol-cut dust (Low)
**Location:** `PawasaveLend.sol` — `liquidate()` computes `seizeAmount = _cngnToCollateral(...)`, then
`protocolCut = seizeAmount * liquidationProtocolFeeMantissa / BASE`, `liquidatorSeize = seizeAmount - protocolCut`,
and transfers both, decrementing collateral by the full `seizeAmount`.

**Description:** CEI ordering is correct (state updates — `_updateBorrowBalance`, `totalBorrows`,
`dueDate`, collateral decrement — happen before the external `safeTransfer`s, and `nonReentrant` is
present). The only nit: rounding in `_cngnToCollateral` and the protocol-cut split can leave 1-unit
dust and, for very small `repayAmount`, `protocolCut` can round to 0 (liquidator takes the whole
bonus). No invariant break.

**Impact:** Low — negligible dust; protocol occasionally forgoes a sub-unit cut.

**Recommendation:** Acceptable as-is; optionally add a minimum-repay floor to `liquidate` mirroring
`CreditLine.MIN_REPAY` to avoid dust-liquidation griefing.

## 6. Oracle & Keepers

### V3-OK-01 — Single hot-EOA keeper, single primary rate source, no on-chain cross-check (High)
**Location:** `scripts/oracle-keeper.ts` (`fetchNgnUsdRate` — Flint primary, `open.er-api.com` fallback;
`ORACLE_KEEPER_PRIVATE_KEY` signer), consumed by `PriceOracle.setPrice`.

**Description:** All USDC/USDT collateral pricing flows from one keeper EOA reading one primary source
(Flint) with a free-tier public fallback, then pushing to the oracle. There is no on-chain Chainlink
NGN/USD cross-check (the contract comment acknowledges this is future work). The keeper skips updates
within 0.5% of the on-chain value.

**Impact:** High. A wrong/manipulated rate from the single primary source (or a compromised keeper key)
directly moves collateral valuation, enabling either unfair liquidations (price too low) or
under-collateralized borrows (price too high), bounded only by the 25% deviation breaker (and only for
non-`forceSetPrice` writes, and only after a first value exists).

**Recommendation:** (1) Add a **second independent rate source** and require agreement within a band
before pushing; (2) integrate an **on-chain Chainlink** NGN/USD (or USD/USDC sanity) feed as a
cross-check once available on Base; (3) move the keeper key to KMS/HSM (see V3-OPS-01); (4) set a
per-token `minPrice` floor for every collateral so a near-zero first write is impossible.

### V3-OK-02 — 1-hour staleness window (Medium)
**Location:** `PriceOracle.sol` — `MAX_PRICE_AGE = 1 hours`, enforced by `getPrice` and
`collateralToCngn`.

**Description:** Prices up to one hour old are accepted as fresh. In a fast NGN devaluation or USDC
depeg, a borrower could be liquidated on a stale-high or stale-low price, or borrow against stale
collateral, for up to an hour.

**Impact:** Medium — stale-price liquidation/borrow within the window.

**Recommendation:** Tighten `MAX_PRICE_AGE` (e.g. 15–20 min) to match a sub-hourly keeper cadence, and
ensure the keeper schedule is comfortably inside the window with alerting on missed updates.

### V3-OK-03 — Liquidation keeper approves MaxUint256 and scans by event lookback (Low)
**Location:** `frontend/src/app/api/cron/liquidate/route.ts` — `cngn.approve(lendAddr, ethers.MaxUint256)`,
borrower discovery via `Borrowed` events over `LIQUIDATION_LOOKBACK_BLOCKS` (default 200k).

**Description:** The keeper grants an unlimited cNGN allowance to the lend pool and enumerates
borrowers by scanning a recent block window. A borrower whose `Borrowed` event predates the lookback
window would be missed by this backstop (third-party liquidators still cover them). Unlimited approval
means a lend-pool compromise could pull the keeper's cNGN.

**Impact:** Low — the pool is trusted and the keeper holds limited cNGN, but unbounded approval is
unnecessary standing risk and the lookback can silently skip old positions.

**Recommendation:** Approve only the needed amount per run (or a bounded standing allowance), and track
active borrowers in the DB rather than relying solely on a fixed event lookback.

## 7. Operational Secrets & Key Custody

### V3-OPS-01 — Hot keeper/custody keys not yet on KMS/HSM (High) — re-raise of FIND-SC-21 / FIND-3P-05/06
**Location:** `frontend/src/lib/secrets.ts` (resolves `CUSTODY_PRIVATE_KEY`,
`ORACLE_KEEPER_PRIVATE_KEY`, `LIQUIDATION_KEEPER_PRIVATE_KEY`, `DEPOSIT_WALLET_MNEMONIC`, etc.),
`frontend/src/lib/custody.ts` (`getSigner` builds an `ethers.Wallet` from the raw key),
`scripts/oracle-keeper.ts`, `frontend/src/app/api/cron/liquidate/route.ts`.

**Description:** Secrets moved from plain Vercel env to AWS Secrets Manager (good, V2-HIGH-02), but the
custody wallet — which `custody.ts` documents as holding **all user funds** — and both keepers still
sign from **raw private keys loaded into process memory**. `getSecret` returns the key material to
JS. KMS/HSM signing (where the key never leaves the HSM) is still open per FIND-SC-21.

**Re-raise justification:** Prior trackers mark this "in progress / needs your KMS account". It is
re-raised because the custody wallet is a **single hot key over all user funds** and remains the
highest-value operational target; it should gate scaling past beta.

**Impact:** High — a server-side compromise (RCE, dependency supply-chain, leaked AWS creds scoped to
the secret) exposes the key and thus all custody funds and oracle/liquidation authority.

**Recommendation:** Move custody + both keeper signers to **KMS/HSM-backed signing** (AWS KMS
`sign`, or a threshold signer). Sweep-on-receipt (FIND-3P-06) already limits mnemonic-leak blast
radius; do the same conceptually for custody by keeping hot balances minimal and settling to a
cold/Safe address.

### V3-OPS-02 — Exposed deployer key still in Safe signer context (High) — re-raise of FIND-3P-05
**Location:** `.env.example` lines 33–36 explicitly state the deployer key is **considered exposed** and
must be rotated out of the signer set; `scripts/transfer-ownership.ts` performs the Safe migration.

**Description:** The current `DEPLOYER_PRIVATE_KEY` is documented as exposed. Ownership moved to the
Safe, but the trackers note the exposed deployer key still needs removing from the signer set
(→ hardware wallet).

**Re-raise justification:** Marked "in progress" in v1 tracker; re-raised because until the exposed key
is removed, the Safe's effective security is only as good as the *other* signers, and an exposed
signer materially weakens a 2-of-3.

**Impact:** High — an exposed Safe signer reduces the practical threshold and, combined with one more
signer compromise, yields full owner control (see V3-SC-01).

**Recommendation:** Rotate the exposed deployer key **out of the Safe signer set** and replace it with a
hardware-wallet signer; verify the resulting 2-of-3 has no exposed member; do this in the v3 window.

### V3-OPS-03 — RSA private key written to working directory (Medium)
**Location:** `generateKeys.js` (`fs.writeFileSync('private_key.pem', privateKey)`), `sign-xend.js`
(reads a PEM from an arbitrary path and signs).

**Description:** `generateKeys.js` writes an unencrypted RSA private key (`private_key.pem`) into the
current working directory. If run inside the repo tree it risks accidental commit; the key is
unencrypted at rest.

**Impact:** Medium — accidental commit or backup capture of `private_key.pem` leaks the Xend signing
key. (`.gitignore` should be verified to cover it.)

**Recommendation:** Generate keys into a path outside the repo, encrypt the private key at rest (or
generate directly in a secrets manager / KMS), print a reminder, and confirm `*.pem` is git-ignored.
Treat `sign-xend.js` as a local-only dev helper (document it as such).

### V3-OPS-04 — Secrets Manager failure silently falls back to env / stale cache (Low)
**Location:** `frontend/src/lib/secrets.ts` — `loadBundle` catches errors and returns `cache?.values ?? {}`,
then `getSecret` falls back to `process.env[name]`.

**Description:** On a Secrets Manager outage the resolver silently serves the last-good cache and then
`process.env`. During a **key-rotation event** this can keep an old key alive longer than intended if a
transient error masks the new fetch, partially undermining the fast-rotation goal of V2-HIGH-02.

**Impact:** Low — availability-favoring design that can extend the lifetime of a rotated-out key under
error conditions.

**Recommendation:** Distinguish "not configured" (env-only mode) from "configured but failing" — for the
latter, prefer failing closed for **high-value keys** (custody/keeper) rather than falling back to a
possibly-stale value, and emit a loud alert. Keep the graceful path only for low-sensitivity config.

## 8. strails-relay

### V3-RL-01 — Over-broad authenticated proxy of a key-bearing upstream (Medium)
**Location:** `strails-relay/index.js` — after `secretOk`, `const path = (req.url||'/').split('?')[0]`,
validated only by `/^\/[a-zA-Z0-9/_-]*$/`, then forwarded verbatim to `${BASE}${path}` with any method
and the injected `x-api-key`.

**Description:** The relay is correctly narrow in one sense (only forwards to the configured Strails base
URL, constant-time secret compare, path regex blocks traversal), but it forwards **any path and any
method** for any caller holding the shared secret, injecting the Strails API key. There is no per-route
allowlist. It is effectively a full authenticated proxy to a key-bearing upstream.

**Impact:** Medium — if the `RELAY_SECRET` leaks (it lives in Vercel env, a broader surface than the
relay host), an attacker gains full API access to Strails via the relay, across every Strails endpoint,
not just the ones PawaSave uses.

**Recommendation:** Add a **per-route + per-method allowlist** (only the specific Strails paths/verbs the
app actually calls). Rotate `RELAY_SECRET` on a schedule, keep it out of the broad Vercel env if
possible, and add basic rate limiting and request logging on the relay host.

## 9. Supabase Backend / RPC & RLS

### V3-DB-01 — Default-open PostgREST grant model on SECURITY DEFINER money RPCs (High, root cause)
**Location:** `supabase/migrations/077_revoke_service_only_rpcs.sql`,
`supabase/migrations/078_revoke_anon_execute.sql`.

**Description:** The migrations document that **69 of 97** public functions were executable by `anon`
(the published client key = anyone on the internet), including money-movers like `credit_wallet`,
`debit_wallet`, `withdraw_cngn_pool`, `withdraw_vault_atomic`, `create_loan`, `place_equity_order`,
`finalize_kyc`, and `set_deposit_address`. Because these are `SECURITY DEFINER` and their guard is
`IF auth.uid() IS NOT NULL AND auth.uid() != p_user_id`, an **anonymous caller (auth.uid() = NULL)
skips the check** and can name any victim via `p_user_id`. The fix was applied **out-of-band as
hotfixes** and then codified in `077`/`078`.

**Impact:** High. Pre-hotfix this was a critical unauthenticated fund-theft / KYC-bypass surface
(the migration notes no evidence of exploitation). The root cause — a **default-open grant posture**
where new `SECURITY DEFINER` functions inherit `PUBLIC`/`anon EXECUTE` — persists as a class of bug.

**Recommendation:** Adopt a **default-deny posture**: `ALTER DEFAULT PRIVILEGES ... REVOKE EXECUTE ON
FUNCTIONS FROM PUBLIC`, and grant `authenticated`/`service_role` explicitly per function. Verify the
`078` "Group B" functions that stay `authenticated` truly need it.

### V3-DB-02 — `auth.uid() IS NOT NULL AND != p_user_id` guard is a no-op for anon (High)
**Location:** the authorization pattern quoted in both migrations' headers.

**Description:** The guard pattern short-circuits for anonymous callers. Any DEFINER function still
reachable by `anon` that uses this pattern is effectively unauthenticated for a supplied `p_user_id`.
`078` closed client access for the known money-movers, but the **pattern itself is the defect** and may
recur in functions not enumerated in these two migrations.

**Impact:** High wherever it recurs — unauthenticated action against an arbitrary account.

**Recommendation:** Replace the pattern everywhere with a **hard equality that fails closed**:
`IF auth.uid() IS NULL OR auth.uid() != p_user_id THEN RAISE`. Better still, **stop passing `p_user_id`**
for user-scoped actions and derive the user from `auth.uid()` inside the function. Audit all
`SECURITY DEFINER` functions for the old pattern.

### V3-DB-03 — No default-deny / CI lint for new DEFINER functions (Medium)
**Location:** repo-wide (absence of a migration lint / CI check).

**Description:** Nothing prevents a future migration from creating a `SECURITY DEFINER` function that is
again `PUBLIC`/`anon`-executable. The `077`/`078` cleanup is point-in-time.

**Impact:** Medium — regression risk; the same class of hole can reopen silently.

**Recommendation:** Add a **CI check** that fails if any `public` `SECURITY DEFINER` function holds
`EXECUTE` for `PUBLIC` or `anon` (query `information_schema.role_routine_grants` in a test DB spun from
migrations), plus a schema-lint enforcing the `ALTER DEFAULT PRIVILEGES` revoke is present.

## 10. Frontend & Money-Flow Backend

> Verified-safe (no finding): the `flipeet-webhook` completed/failed branches use a **guarded status
> claim** (`.eq('status','pending')` on the update) so concurrent retries can't double-credit or
> double-refund; the webhook token compare uses a length check + `timingSafeEqual`; `admin/verify`
> uses `timingSafeEqual` + a DB-backed IP lockout; `proxy_transfer`/`get_proxy_transfers` are
> admin-gated server-side. The refund micro-unit math (`amount_kobo * 10_000` = kobo→cNGN micro) and
> the deposit `ngnToCngnMicro` path are consistent with the 1 NGN = 1 cNGN peg.

### V3-BE-01 — Webhook auth by secret-token-in-URL (Medium)
**Location:** `frontend/src/app/api/flipeet-webhook/route.ts` — reads the token from
`request.nextUrl.searchParams.get('token')` (or `x-webhook-token`); `FIND-3P-01` notes Flipeet does not
sign callbacks.

**Description:** Authenticity rests on a bearer token embedded in the callback URL handed to Flipeet.
Tokens in URLs leak into provider logs, proxy logs, referrers, and error trackers more readily than
signed request bodies. This is the best available given the provider limitation, but remains weaker
than HMAC request signing.

**Impact:** Medium — token leakage would let an attacker forge completed/failed webhooks, driving
credits/refunds (bounded by the pending-row guard and per-transaction matching).

**Recommendation:** Prefer the `x-webhook-token` **header** over the query param and stop passing the
token in the URL where the provider supports headers; rotate `FLIPEET_WEBHOOK_TOKEN` regularly; scope
credits strictly to matched pending rows (already done) and alert on token-auth failures.

### V3-BE-02 — Auto-routing unmatched deposits via provider-supplied id (Medium)
**Location:** `frontend/src/app/api/flipeet-webhook/route.ts` — when no pending tx matches, it reads
`beneficiaryId` from `data.beneficiary.wallet_address/id`/`data.memberId` and calls
`get_user_for_proxy_member` then `process_proxy_deposit` to credit that user.

**Description:** For unmatched deposits the handler trusts a **provider-supplied beneficiary identifier**
to select which user to credit, and credits `amountNaira` taken from the webhook body. The amount is
bounded by `process_proxy_deposit`'s floor/cap (V2-MED-02), but the routing and amount both originate
from provider input authenticated only by the URL token.

**Impact:** Medium — if the token leaks (V3-BE-01), an attacker could credit an arbitrary mapped user an
attacker-chosen amount (within the cap), or map credits to unintended accounts.

**Recommendation:** Only auto-route when the beneficiary id maps to a **known, expected pending
intent** for that user/amount; log and quarantine unmatched deposits for manual review instead of
auto-crediting; re-verify the amount against an expected value where possible.

### V3-BE-03 — Off-ramp debit→provider→refund is multi-step, not atomic (Medium)
**Location:** `frontend/src/app/api/ramp/route.ts` — `runXend` off-ramp: `maybeDebitForWithdrawal` →
insert `withdrawal` row → `proxyFundsTransfer(CREDIT)` → `proxyCryptoToFiatTransfer` → on error, refund
via `proxyFundsTransfer(DEBIT)` + credit back.

**Description:** The off-ramp debits the user, then performs multiple provider calls, then compensates on
failure. This is a saga, not an atomic transaction: a crash between the debit and the compensating
refund (serverless timeout, provider partial success, process kill) can leave a user debited without a
completed transfer and without an executed refund. The `custody`/lease helpers and the reconciler
mitigate but do not eliminate the window.

**Impact:** Medium — transient user-fund limbo requiring reconciliation; not a theft vector but a
correctness/consumer-trust risk under partial failure.

**Recommendation:** Ensure a **durable reconciliation cron** (like the V2-MED-06 lend-supply retry queue)
covers every off-ramp state transition idempotently, keyed by `reference`; record an explicit
"debited-awaiting-provider" state so a crash is recoverable to either completion or refund exactly once.

### V3-BE-04 — Admin auth single password, no MFA (Low, re-raise of FIND-AUTH-03 / FIND-FE-01)
**Location:** `frontend/src/app/api/admin/verify/route.ts` — single `ADMIN_PASSWORD`, `timingSafeEqual`,
IP lockout, httpOnly session cookie.

**Description:** Admin access is a single shared password (hardened with constant-time compare, IP
lockout, and an HMAC session cookie). No MFA. Prior trackers mark this **accepted-by-decision**.

**Re-raise justification:** Re-raised only as **should-be-revisited before scaling past beta** — the
admin surface can trigger `proxy_transfer` and read cross-user data, so a single-factor credential is a
meaningful residual risk at scale.

**Impact:** Low today (mitigated), rising with scale.

**Recommendation:** Add MFA (TOTP/WebAuthn) to admin login before broad launch; consider per-admin
identities rather than one shared password.

## 11. CI/CD & Testing

### V3-CI-01 — Slither runs `fail-on: none` (Medium)
**Location:** `.github/workflows/ci.yml` — the `slither` job sets `continue-on-error: true` and the
action `fail-on: none`.

**Description:** Static analysis is purely informational; a newly introduced high/medium Slither finding
would not fail CI. For contracts holding user funds this is under-gated.

**Impact:** Medium — regressions in contract safety could merge unnoticed by CI.

**Recommendation:** Set Slither to **gate on high (and ideally medium)** severity for the contracts,
with a reviewed, checked-in triage/allowlist for accepted findings.

### V3-CI-02 — No fuzz/invariant tests for lending math; no frontend unit tests (Low, re-raise of V2-DEP-03)
**Location:** contract tests are TS/Hardhat unit tests (`test/*.ts`); no Foundry/Echidna
invariant/fuzz suite; frontend has no unit tests (V2-DEP-03 open).

**Description:** The interest-accrual, exchange-rate, liquidation, and vault↔strategy accounting are
exactly the areas where property/invariant testing catches rounding and economic edge cases that
example-based unit tests miss.

**Impact:** Low-to-Medium confidence gap on the highest-value math.

**Recommendation:** Add **Foundry invariant/fuzz tests** for: `totalPoolAssets` invariance across
borrow→repay cycles, share-price monotonicity under harvest, exchange-rate behavior under bad debt
(supports V3-SC-05), and liquidation seize/close-factor bounds. Add minimal frontend unit tests for the
money-math helpers (`ramp-rate`, micro-unit conversions).

## 12. Deployed vs. source drift

The **7d stack is live on Base mainnet** (addresses in `SECURITY_AUDIT_REMEDIATION.md` §7d:
IRM `0x60Dd…4Ddd`, Oracle `0x58a1…3BdF`, Lend `0x14c5…2B93`, Strategy `0xB9f6…dc81`,
Vault `0x000B…b9E2`; CreditLine redeployed to `0x5056…1B13`, old `0x723C…3F06` abandoned). The
following v2 fixes **exist in the reviewed source but are NOT live on that on-chain stack** — they take
effect only on a **v3 redeploy** (the oracle is `immutable` in `PawasaveLend`, so IRM/oracle/lend/
vault/strategy must be redeployed together):

- **V2-SC-09** — `PriceOracle.setMaxDeviation` 50% cap (`MAX_DEVIATION_CEILING_BPS`) so the breaker
  can't be disabled. *Not live.*
- **V2-SC-10** — `KeeperUpdated` event on `setKeeper`. *Not live.*
- **V2-MED-01** — Vault `_withdraw` acting on each strategy's **actual returned amount** (top-up from
  fallback, revert on genuine shortfall). *Not live.*
- **PriceOracle per-token `minPrice` floor** (applies even to `forceSetPrice`) — the guard that makes
  V3-SC-01 / V3-OK-01 materially safer. *Not live.*
- **V2-SC-02/04** — `PawasaveLendStrategy.setPaused()` to block new deposits while keeping
  withdraw/harvest open. *Not live.*

Also outstanding operationally (not on-chain drift but gating): **V2-INFRA-05** — the Safe must call
`acceptOwnership()` on the `Ownable2Step` vault; Lend/Oracle/CreditLine ownership already transferred.

**Gating action:** a **v3 redeploy + re-audit** is required before these source fixes protect real
funds, and `CreditLine` must be audited before it is deployed. Until then, the live protocol behaves per
the *pre-fix* source for these items.

## 13. Recommended upgrades & changes (prioritized)

### P0 — do before scaling past the beta cohort
On-chain (needs redeploy):
- [ ] Bundle the v2 "fixed-in-source" changes (V2-SC-09, V2-SC-10, V2-MED-01, oracle `minPrice` floor,
      V2-SC-02/04) into a **v3 stack redeploy**, then **re-audit** (§12).
- [ ] Set a **per-token `minPrice` floor for every accepted collateral** at redeploy (bounds
      `forceSetPrice`, V3-SC-01/V3-OK-01).
- [ ] Audit `PawasaveCreditLine` before any deploy; keep it under-funded / just-in-time (V3-SC-03).

Operational (keys / KMS / Safe):
- [ ] Move **custody + oracle + liquidation signers to KMS/HSM** (V3-OPS-01).
- [ ] **Rotate the exposed deployer key out of the Safe signer set** and verify the 2-of-3 (V3-OPS-02).
- [ ] Call **`acceptOwnership()`** on the vault from the Safe (V2-INFRA-05).

Off-chain (backend):
- [ ] Adopt **default-deny** `ALTER DEFAULT PRIVILEGES` on functions and replace the
      `auth.uid() IS NOT NULL AND != p_user_id` pattern with a **fail-closed equality** across all
      `SECURITY DEFINER` functions (V3-DB-01, V3-DB-02).

### P1 — before broad launch
On-chain:
- [ ] Decide the **overdue-but-healthy liquidation bonus** policy; consider a reduced/zero bonus for the
      maturity path (V3-SC-02).
- [ ] Standardize on **`Ownable2Step`** for Lend/Oracle/CreditLine/Strategy (V3-SC-07).
- [ ] Add a **timelock in front of the Safe** for parameter changes (V3-SC-01).
- [ ] Tighten `MAX_PRICE_AGE` to match keeper cadence; add a **second rate source + Chainlink
      cross-check** (V3-OK-01, V3-OK-02).

Operational:
- [ ] Add **per-route/method allowlist**, rate limiting, and secret rotation to `strails-relay` (V3-RL-01).
- [ ] Generate RSA keys outside the repo, encrypted at rest; confirm `*.pem` is git-ignored (V3-OPS-03).
- [ ] Fail closed on Secrets Manager errors for high-value keys (V3-OPS-04).

Off-chain:
- [ ] Add a **CI lint** that fails on any `PUBLIC`/`anon` EXECUTE on `SECURITY DEFINER` functions (V3-DB-03).
- [ ] Prefer header over URL for the webhook token; quarantine/verify **auto-routed deposits**
      (V3-BE-01, V3-BE-02).
- [ ] Make the **off-ramp saga fully idempotent + reconciled** end-to-end (V3-BE-03).
- [ ] Gate **Slither on high/medium** in CI (V3-CI-01).
- [ ] Add **MFA** to admin login (V3-BE-04).

### P2 — hardening / defense-in-depth
On-chain:
- [ ] Zero `principal`/`dueDate` on max repay to avoid dust-open loans (V3-SC-04).
- [ ] Add a `MIN_REPAY`-style floor to `liquidate` (V3-SC-08).
- [ ] Bound the liquidation keeper's cNGN approval instead of `MaxUint256` (V3-OK-03).

Off-chain / testing:
- [ ] Add **Foundry invariant/fuzz** tests for lending math and vault↔strategy under bad debt
      (V3-SC-05, V3-CI-02).
- [ ] Add frontend unit tests for money-math helpers (V3-CI-02).
- [ ] Monitor/alert on accrual-cron and keeper liveness (V3-SC-06, V3-OK-02).

---

*This report is a source-level review and does not constitute a guarantee of security. It excludes
items already remediated in `SECURITY_AUDIT_REMEDIATION.md` and `AUDIT_V2_REMEDIATION.md`; re-raised
items are tagged with their prior IDs. A v3 redeploy + re-audit and the KMS/Safe operational items are
the gating actions before scaling past the beta cohort.*
