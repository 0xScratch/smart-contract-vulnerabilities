# reNFT — ERC1155 Hijack via Reentrancy / TOCTOU (rentedAssets)

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-02-renft-mitigation-findings/issues/27)
* **Affected Contracts**: [Storage.sol](https://github.com/re-nft/_v3_.smart-contracts/blob/97e5753e5398da65d3d26735e9d6439c757720f5/src/modules/Storage.sol)
* **Vulnerability Type**: Reentrancy / TOCTOU (time-of-check → time-of-use) / State-ordering bug

## Summary (one paragraph)

An attacker creates a *self-lend* where the lender is a malicious contract the attacker controls and the borrower is their rental safe. When `stopRent()` runs for that malicious order the protocol **removes the rental bookkeeping (`rentedAssets`) before completing token reclaims**. The malicious lender's `onERC1155Received` (or `onERC721Received`) callback triggers and makes the Gnosis Safe execute a pre-signed `transferFrom` that moves most tokens out while guard checks see the already-reduced `rentedAssets`. Because storage was updated early, the guard's arithmetic allows the draining. The reclaim loop then finishes and transfers the remainder — leaving the rental safe drained (or partially drained) and the legitimate rental irrecoverable. Result: attacker hijacks ERC1155 tokens and bricks rentals.

## A better explanation (with a simple numbered example)

### Intended invariant

At any point during reclaiming, the safe should always hold at least `rentedAssets[safe, token, id]` tokens of that `(token, id)` so that rentals can be reclaimed correctly.

### Concrete numbers (simplified)

1. Alice lends attacker 100 tokens (ERC1155 id=5).

   * `rentedAssets[safe,id]=100`, `safe.balance=100`.
2. Attacker self-lends another 100 via malicious order (lender == malicious contract).

   * After malicious order begins: `rentedAssets=200`, `safe.balance=200`.
3. Attacker calls `stopRent()` on malicious order. **Bug**: `Storage::removeRentals()` runs **first**, subtracting 100 → `rentedAssets=100`, but `safe.balance` still = 200.
4. Reclaimer transfers the first offer item (1 token) to the malicious lender. `safe.balance` becomes 199. This triggers `onERC1155Received` in the malicious lender.
5. In the callback the malicious lender executes a pre-signed Safe `execTransaction` that requests removal of 99 tokens from the safe.

   * Guard computes: `rentedAmount = STORE.isRentedOut(safe,token,id) = 100` (already reduced), `safeBalance = 199`, `remainingBalance = 199 − 99 = 100`.
   * Since `rentedAmount > remainingBalance` is false (100 > 100 ? no), guard allows the transfer.
6. Attacker receives 99 (safe now 100). Reclaimer then transfers the second offer item (99) to the lender (path may bypass same guard), leaving safe = 1. Attacker ends with 199; legitimate lender's 100 is unrecoverable.

### Core logical failure

**Storage was updated (rentedAssets decreased) before external interactions finished**, so checks that relied on rentedAssets observed an incorrect (already-applied) state while the safe's balance could still be manipulated by reentrancy.

## Key vulnerable code locations / patterns (what to look for)

* `Storage::addRentals()` / `Storage::removeRentals()` — these update `rentedAssets` and `orders`. The problematic ordering is `removeRentals()` being called before reclaim finishes.
* `Stop::stopRent()` / `Stop::_reclaimRentedItems()` → calls into `Reclaimer::reclaimRentalOrder()` which performs `safeTransferFrom` calls that trigger token callbacks.
* Gnosis Safe guard code path (the guard that runs on Safe `execTransaction`) — it reads `STORE.isRentedOut(safe, token, tokenId)` and `IERC1155(token).balanceOf(safe, tokenId)` and compares `rentedAmount` and `remainingBalance`. Because `rentedAmount` was reduced already, the check can pass even though tokens are being drained mid-operation.
* Malicious primitive used in PoC: a lender contract implementing `onERC1155Received` which calls back into the safe via `execTransaction` using a pre-signed signature (provided by the attacker).

(Refer to the PoC `Exploit.sol` for a concrete test harness that sets up: legitimate rental, malicious self-rental, pre-signed safe tx, time warp, then calls `stop.stopRent(maliciousRentalOrder)` to demonstrate the exploit.)

## Short, focused remediation options (practical & commonly recommended)

### 1) Move storage updates AFTER successful reclaim (preferred conceptual fix)

* Don't call `removeRentals()` or decrement `rentedAssets` until **after** the reclaim loop completes successfully. That way `rentedAssets` still reflects the true minimum while reclaims and callbacks run, preventing the TOCTOU.
* Implementation note: because reclaims call external code, you must either:

  * Use a reentrancy guard that blocks callbacks from making balance-altering Safe execs while reclaiming, or
  * Use a two-phase approach: mark order as `reclaiming` (a flag) and ensure guard logic treats `reclaiming` orders as still counted.

### 2) Block or limit the primitive that enables reentrancy (defense in depth)

* Temporarily or permanently **disallow contract lenders** (or require them to be whitelisted/audited) OR disallow self-lend orders where lender is a contract and fulfiller equals borrower's safe. This removes the callback vector.
* Or detect and reject self-lend orders at creation time.

### 3) Add a reclaim-time invariant check / reentrancy lock

* Set a per-safe `inReclaim` boolean while reclaim runs. The guard must reject `execTransaction`/transfer attempts that originate from the same safe during `inReclaim`. Clear the flag only after finalizing storage changes.
* Alternatively, have the reclaimer capture `preReclaimRented = STORE.isRentedOut(...)` and `preReclaimBalance = balanceOf(...)` and verify after each token transfer that invariants still hold; revert if tampered.

### 4) Unify guard checks and transfer paths

* Ensure reclaim transfers and Safe exec transfers use the same guard/enforcement path, and ensure that guard uses conservative values (e.g., include pending reclaim amounts) so there's no bypass due to path differences.

> The simplest safe immediate patch for producers is to **prevent contract lenders / self-lends** or **add an in-progress reclaim lock** — lower risk and quick. Proper fix is to ensure bookkeeping reflects pre-reclaim invariants until all external calls finish.

## Pattern recognition / lessons learned

* **TOCTOU / effect-before-interaction**: updating critical bookkeeping before finishing external calls creates windows attackers can exploit via callbacks.
* **Callbacks + multisig (Gnosis) interactions are dangerous**: token transfer callbacks (ERC-1155/ERC-721) plus pre-signed multisig execs enable attackers to reorder flows and move funds mid-operation.
* **Don't treat `rentedAssets` as authoritative if it can be changed prior to finishing external operations**. Either keep it authoritative until operation completes, or design guards to include pending state.
* **Self-lend is a risky feature** — when lender and borrower are related and lender is a contract, the attacker gains callback + signature primitives that defeat many invariants.

## TL;DR

`removeRentals()` updates the on-chain rented counters too early. A malicious lender contract uses ERC-1155 callbacks + a pre-signed Gnosis Safe exec to remove tokens during reclaim while the guard reads the already-reduced `rentedAssets`, allowing theft. Fix by keeping bookkeeping authoritative during reclaim (or locking reentrancy / disallowing unsafe self-lends).
