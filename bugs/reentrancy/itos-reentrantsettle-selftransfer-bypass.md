# Self-Transfer Settlement Bypass in `reentrantSettle`

* **Severity**: High
* **Source**: [Pashov Audit Group](https://github.com/pashov/audits/blob/master/team/md/Itos-security-review_2025-05-24.md#h-01-payers-exploit-reentrantsettle-to-bypass-payments-with-self-transfers)
* **Affected Contract**: [RFT.sol](https://github.com/itos-finance/Commons/blob/c92003b0c8d91262c1e2f69f17ddd9aa581eb21a/src/Util/RFT.sol)
* **Vulnerability Type**: Reentrancy / Balance Accounting Manipulation

## Summary

The `reentrantSettle` function in Itos Finance allows nested settlement calls and uses a **cumulative delta-based accounting system** to track expected token balance changes.

Because the function does not prevent **self-transfers** (when the `payer == address(this)`), an attacker can cause the contract to "transfer tokens to itself" during a nested call. On ERC-20s, self-transfers leave balances unchanged, but `transact.delta` is still adjusted. This lets the attacker cancel out expected inflows with fake outflows, causing the final balance check to pass even though no real tokens were received.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Call `reentrantSettle`**: Contract records expected token deltas in `transactionStorage`.

   * Positive delta → contract expects to **receive** tokens.
   * Negative delta → contract expects to **send** tokens.
2. **Transfers**: Contract sends what it owes; counterparties send tokens in return.
3. **Final Check**: After all nested calls, contract ensures:

   ```javascript
   finalBalance >= startBalance + delta
   ```

   for each token. This should guarantee that expected inflows happened.

### What Actually Happens (Bug)

* Attacker sets `payer = address(this)` (victim itself).
* A nested call triggers a **self-transfer** of tokens. On ERC-20, self-transfer does not change balances.
* But bookkeeping applies `delta -= amount`, cancelling out a prior `+amount`.
* Final check compares equal numbers → passes, even though victim never received the tokens.

### Why This Matters

* The protocol's settlement system can be **tricked into believing a payment succeeded**.
* Attackers can obtain services or assets **without actually paying**.
* Since the final balance check passes, the protocol has no way to detect the fraud post-facto.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Victim has 100 USDC. `startBalance = 100`.
* **Alice expects payment**: Outer `reentrantSettle` requests `+10 USDC`. So `delta = +10`.
* **Mallory's malicious payer callback**:
  Calls nested:

  ```solidity
  reentrantSettle(payer = address(this), tokens=[USDC], balanceChanges=[-10]);
  ```

  * Victim executes self-transfer of 10 USDC (balance unchanged at 100).
  * `delta` becomes `0`.
* **Final check**:

  ```javascript
  expected = 100 + 0 = 100
  finalBalance = 100
  ```

  → passes. Alice never actually got her 10 USDC.

> **Analogy**: Imagine a store's system expects \$10 incoming. The attacker makes the store's cashier issue a fake \$10 refund to itself. Cash in the register doesn't change, but the ledger cancels the debt. The system believes the payment cleared even though no one paid.

## Vulnerable Code Reference

### 1) Cumulative delta logic in `reentrantSettle`

```solidity
// After processing each balance change
int256 newDelta = transact.delta[token] + change;
transact.delta[token] = newDelta;
```

### 2) Self-transfer executed when `payer == address(this)`

```solidity
if (change < 0) {
    uint256 absChange = uint256(-change);
    TransferHelper.safeTransfer(token, payer, absChange); 
    // no-op if payer == address(this) → balance unchanged
}
```

### 3) Final check trusts delta arithmetic

```solidity
uint256 expected = transact.startBalance[token] + uint256(transact.delta[token]);
require(balance >= expected, "underfunded");
```

## Recommended Mitigation

1. **Block self-transfers (primary fix)**

    ```solidity
    require(payer != address(this), "Self-settlement not allowed");
    ```

2. **Stricter validation on balance changes**:
   Disallow negative `change` if `payer == address(this)`.

3. **Balance check hardening**:
   Compare actual *observed* incoming transfers per payer rather than trusting cumulative deltas.

4. **Add invariant tests**:
   Ensure that "expected positive inflow" cannot be erased by a nested negative change involving `address(this)`.

## Pattern Recognition Notes

* **Self-Transfer Abuse**: ERC-20 self-transfers often leave balances unchanged but still trigger events. This can desynchronize accounting systems that rely on deltas.
* **Nested Settlement Risk**: Allowing reentrant bookkeeping requires extra safeguards — cumulative deltas are fragile when manipulated in reentrant flows.
* **Sentinel Mismatch**: `delta == 0` is treated as "all balanced," but can be maliciously forced via canceling operations.
* **Reentrancy Vector**: Callback into the victim during settlement opened the path to inject malicious state changes.
* **Mitigation Pattern**: Early rejects (e.g., `payer != this`) are simpler and safer than trying to handle it downstream.

### Quick Recall (TL;DR)

* **Bug**: `reentrantSettle` allows self-transfers; attacker cancels expected inflows with no-op outflows.
* **Impact**: Contract falsely thinks settlement succeeded → attacker skips payment.
* **Fix**: Disallow `payer == address(this)` and enforce stronger balance validation.
