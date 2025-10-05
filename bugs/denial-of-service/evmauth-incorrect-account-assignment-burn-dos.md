# Incorrect Account Assignment in Token Burning Logic in EVMAuthExpiringERC1155

* **Severity**: High
* **Source**: [Trail of Bits Audit Report - Radius Technology EVMAuth](https://github.com/trailofbits/publications/blob/master/reviews/2025-10-radiustechnology-evmauth-securityreview.pdf)
* **Affected Contract**: [EVMAuthExpiringERC1155.sol](https://github.com/evmauth/evmauth-core/blob/63835cf772c8b95e6a1bd69cdf6f47834c356eca/src/base/EVMAuthExpiringERC1155.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Logic Error / Incorrect Variable Assignment

## Summary

The `_update` function in `EVMAuthExpiringERC1155` is responsible for synchronizing token expiration and group data during **minting**, **burning**, and **transferring**.
However, in the burning branch, the logic incorrectly sets the `_account` variable to `to` (which is the zero address) instead of `from` (the actual token holder).

As a result, all burn operations attempt to remove tokens from the zero address, which will always fail because the zero address never holds tokens.
This leads to a **permanent Denial of Service for all legitimate burn actions**, effectively preventing revocation or expiration of tokens.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The function `_update(from, to, ids, values)` should:

1. **Minting (`from == 0x0`)**
   → Add token balance and record expiration for the receiver `to`.
2. **Burning (`to == 0x0`)**
   → Subtract token balance and clean up metadata for the sender `from`.
3. **Transferring (otherwise)**
   → Move token balance between users.

### Actual Behavior (Bug)

During the **burning** step, the function mistakenly defines:

```solidity
address _account = to; // ❌ incorrect
```

But since `to` equals `address(0)` when burning, the contract effectively runs:

```solidity
_burnGroupBalances(address(0), _id, _amount);
```

That means it tries to burn from the **zero address** — which never holds tokens — causing the operation to **revert** every time.

## Why This Matters

Because burning tokens is essential to revoke expired or invalid authentication tokens, this bug blocks the contract's ability to:

* Remove expired access tokens
* Enforce access revocation for users
* Maintain correct balance and expiration states

In a real-world scenario, this causes **persistent token validity**, allowing users to keep privileges even after expiry or cancellation.

## Concrete Walkthrough (Alice Example)

1. **Setup**: Alice has authentication tokens (`id = 1, amount = 10`) granting her premium service access.
2. **Admin attempts to burn**:
   The admin calls a burn function that triggers `_update(from = Alice, to = 0x0, ...)`.
3. **During execution**:

   ```solidity
   else if (to == address(0)) {
       address _account = to; // becomes 0x0
       _burnGroupBalances(_account, _id, _amount); // tries to burn from 0x0
   }
   ```

4. **Result**: The burn reverts because address(0) has no tokens.
   → Tokens remain in Alice's balance → Alice continues accessing the service indefinitely.

> **Analogy**: It's like a library trying to take back a borrowed book, but the system records the borrower as "nobody," so the return attempt fails. The book stays with the real borrower forever.

## Vulnerable Code Reference

```solidity
// Burning (incorrect)
else if (to == address(0)) {
    address _account = to; // ❌ Should be 'from'
    _burnGroupBalances(_account, _id, _amount);
    _pruneGroups(_account, _id);
}
```

## Recommended Mitigation

1. **Fix the variable assignment**

   ```solidity
   else if (to == address(0)) {
       address _account = from; // ✅ Correct
       _burnGroupBalances(_account, _id, _amount);
       _pruneGroups(_account, _id);
   }
   ```

2. **Add a unit test for burning**

   * Test burning tokens from non-zero addresses.
   * Ensure burn calls revert when `from == 0x0`.

3. **Add functional tests for expiration and revocation**

   * Simulate end-to-end access token revocation.
   * Verify that post-burn, tokens no longer grant permissions.

4. **Consider assertion checks**

   ```solidity
   require(from != address(0) || to != address(0), "Invalid zero address in _update");
   ```

## Pattern Recognition Notes

* **Variable Reuse Error**: Assigning to the wrong variable in similar-looking branches (mint vs burn) is a common copy-paste pitfall.
* **Symmetry Mismatch**: Mint and burn are mirror operations — any change in one should reflect inversely in the other.
* **Zero Address Confusion**: When `address(0)` is used as a sentinel for mint/burn, variable assignments must be double-checked.
* **Access Token Systems**: Bugs in burn or expiration logic often lead to **revocation failure**, allowing unauthorized users continued access.
* **Unit Test Importance**: Lack of burn path testing led to this oversight — always test all lifecycle paths (mint, burn, transfer).

### Quick Recall (TL;DR)

| Operation | Intended Account | Actual Account           | Result  |
| --------- | ---------------- | ------------------------ | ------- |
| Minting   | `to`             | ✅ Correct                | Works   |
| Burning   | `from`           | ❌ Incorrect (`to = 0x0`) | Reverts |
| Transfer  | `from` & `to`    | ✅ Correct                | Works   |

**Bug:** Burn logic references `to` instead of `from`.
**Impact:** Tokens can never be burned → revocation impossible → privilege persistence.
**Fix:** Use `from` as `_account` in the burning branch and add lifecycle tests.
