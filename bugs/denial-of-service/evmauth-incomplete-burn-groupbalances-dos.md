# Incomplete Burn Handling in `_burnGroupBalances` (EVMAuthExpiringERC1155)

* **Severity**: High
* **Source**: [Trail of Bits Audit Report - Radius Technology EVMAuth](https://github.com/trailofbits/publications/blob/master/reviews/2025-10-radiustechnology-evmauth-securityreview.pdf)
* **Affected Contract**: [EVMAuthExpiringERC1155.sol](https://github.com/evmauth/evmauth-core/blob/63835cf772c8b95e6a1bd69cdf6f47834c356eca/src/base/EVMAuthExpiringERC1155.sol)
* **Vulnerability Type**: Denial of Service / State Inconsistency / Logic Error

## Summary

`_burnGroupBalances` attempts to burn tokens from a user by iterating over their token **groups** (batches with the same expiration) in FIFO order. The function **skips expired groups** and burns only from non-expired batches. However, if expired groups remove a large portion of the nominal balance, the loop may finish with `debt > 0` (meaning not all requested tokens were burned). The function contains **no final validation** that ensures the burn completed. This leads to inconsistencies between the ERC-1155 balance (which may have been reduced) and the internal group tracking (which may not reflect the full burn), causing incorrect future behavior and potential DoS or logic failures.

## A Better Explanation (With Simplified Example)

### Intended Behavior (what should happen)

1. User has tokens split across groups (each group has `balance` and `expiresAt`).
2. When burning `amount` tokens, the code should remove `amount` tokens from the user's **non-expired** groups in FIFO order, and either:

   * fully complete the burn, or
   * revert if there are not enough non-expired tokens.

This ensures ERC-1155 balances and `_group` tracking remain consistent.

### What Actually Happens (bug)

* The function loops through groups and **skips expired groups** (`expiresAt <= now`).
* It reduces `debt` by consuming non-expired groups only.
* When the loop ends, the code **does not check** whether `debt == 0`. If not, the function silently returns with an **incomplete burn**.
* ERC-1155 balance changes (performed elsewhere in the flow) can reflect the burn, but `_group` arrays may not, yielding **divergent state**.

### Why This Matters

* Future operations that rely on `_group` (expiry checks, transfers, further burns, access revocation) can behave incorrectly because `_group` no longer matches the ERC-1155 balance.
* This can cause unexpected reverts, stale access rights, or other DoS scenarios in code using `_group` as the source of truth.
* Silent state corruption is particularly dangerous because it is harder to detect and test for.

### Concrete Walkthrough (Alice)

Initial groups for `Alice` (current time = 150):

* Group A: `{ balance: 3, expiresAt: 100 }` → expired
* Group B: `{ balance: 2, expiresAt: 200 }` → active

Burn request: `burn(4)`

Loop behavior:

1. Group A expired → skipped (no debt reduction).
2. Group B has 2 → burn 2; `debt = 2`.
3. No more groups → loop exits with `debt = 2`.

Result:

* ERC-1155 flow that initiated the burn may mark 4 tokens burned (or the external call completed), but `_group` has only recorded 2 burned.
* **Inconsistent state**: ERC-1155 balance vs `_group` balances differ → future logic breaks.

## Vulnerable Code Reference

Function (excerpt):

```solidity
function _burnGroupBalances(address account, uint256 id, uint256 amount) internal {
    Group[] storage groups = _group[account][id];
    uint256 _now = block.timestamp;
    uint256 debt = amount;
    uint256 i = 0;

    while (i < groups.length && debt > 0) {
        if (groups[i].expiresAt <= _now) {
            i++;
            continue;
        }
        if (groups[i].balance > debt) {
            // Burn partial token group
            groups[i].balance -= debt;
            debt = 0;
        } else {
            // Burn entire token group
            debt -= groups[i].balance;
            groups[i].balance = 0;
        }
        i++;
    }
    // ❌ Missing: require(debt == 0) or other handling if debt > 0
}
```

**Issue:** The function ignores the case where `debt > 0` after iterating non-expired groups — there's no revert, state correction, or explicit handling. This leaves `_group` out of sync with the expected burned amount.

## Recommended Mitigation

### Short term (quick, immediate)

Add an explicit validation at the end of the function so incomplete burns revert and cannot create inconsistent state:

```solidity
// after loop
require(debt == 0, "Incomplete burn: not enough unexpired balance");
```

This prevents silent partial burns and keeps group tracking consistent with external burn semantics.

### Medium term (improve UX & correctness)

* Consider returning the actual amount burned from `_burnGroupBalances` and let the caller decide how to update ERC-1155 state (atomicity can be preserved by ensuring both succeed or revert together).
* Or ensure ERC-1155 balance updates and `_group` updates are always performed atomically from the same transactional context and use checks that guarantee both sides agree before finalizing state.

### Long term (design robustness)

* Redesign burn logic to explicitly account for expired tokens at an earlier stage, e.g. prune expired groups before calculating available balance, or compute `available = sum(non-expired balances)` first and `require(available >= amount)` before any state changes.
* Add comprehensive unit & property tests covering:

  * burning when some groups are expired,
  * burning exactly available non-expired tokens,
  * attempt to burn more than available non-expired tokens (should revert),
  * interactions with `_pruneGroups()` and transfer edge cases.

### Observability / recovery

* Emit an event on failed burn attempts or when incomplete burns are attempted (if choosing to skip rather than revert) to help on-chain debugging and alerting.
* Add an admin/emergency function that can reconcile `_group` arrays with ERC-1155 balances when manual recovery is required (preferably only as last resort).

## Pattern Recognition Notes

* **Silent partial operations**: Functions that modify multiple representations of the same state must ensure all changes complete or they must revert — partial completion is dangerous.
* **Expired-data handling**: When skipping expired entries, code must account explicitly for how that affects operations that rely on current totals; skipping without verification invites state mismatch.
* **Check-before-change vs change-then-verify**: Better to compute and validate preconditions (like `available >= amount`) before mutating state, rather than mutate then check.
* **Atomicity**: Keep related updates (ERC-1155 ledger, `_group` arrays, events) tightly coupled in a single code path or transaction so either all succeed or the whole operation reverts.
* **Test coverage**: Edge cases that combine time (expiry) and state changes are often missed in tests — include them in CI.

### Quick Recall (TL;DR)

* **Bug**: `_burnGroupBalances` skips expired groups and does not verify the burn completed (`debt == 0`), allowing incomplete burns.
* **Impact**: ERC-1155 balances and `_group` tracking can diverge → broken revocation, access checks, and possible DoS or persistent inconsistent state.
* **Fix**: Require `debt == 0` at the end of the function or validate available non-expired balance before mutating state; add tests.
