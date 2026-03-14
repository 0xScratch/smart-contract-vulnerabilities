# Replayable Meta-Transaction via Nonce Rollback on Failed Execution

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-03-rolla-findings/issues/45)
* **Affected Contract**: [EIP712MetaTransaction.sol](https://github.com/code-423n4/2022-03-rolla/blob/efe4a3c1af8d77c5dfb5ba110c3507e67a061bdd/quant-protocol/contracts/utils/EIP712MetaTransaction.sol)
* **Vulnerability Type**: Replay Attack / Nonce Management / Meta-Transaction Design Flaw

## Summary

The `executeMetaTransaction` implementation increments the user's nonce **before executing the underlying call**, but then **reverts the entire transaction if the call fails**.

Because a revert rolls back all state changes, the nonce increment is also reverted. This leaves the nonce **unchanged**, allowing the **same signed meta-transaction to be replayed later** using the same signature.

If the original failure was caused by a **temporary condition** (e.g., insufficient balance or time-dependent state), the transaction may succeed at a later time when conditions change. This can cause **unexpected execution of old signed transactions**, potentially leading to unintended asset transfers or trading actions.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Meta-transactions allow users to sign transactions off-chain and have someone else submit them on-chain.

Typical execution flow:

1. User signs a message containing:

   * intended action
   * nonce
   * parameters
2. Relayer submits the signed message to the contract.
3. Contract verifies the signature and nonce.
4. Contract increments the nonce to prevent replay.
5. Contract executes the requested action.

The nonce mechanism ensures:

```text
Each signature can only be executed once.
```

### What Actually Happens (Bug)

In `executeMetaTransaction`:

1. Signature is verified.
2. Nonce is incremented.
3. The contract executes a low-level call.
4. If the call fails, `require(success)` **reverts the entire transaction**.

Because Solidity reverts roll back **all state changes**, the nonce increment is undone.

This results in:

```text
The same signed meta-transaction remaining valid.
```

Anyone who has the signature can submit it again later.

### Why This Matters

Meta-transactions often control sensitive DeFi operations such as:

* opening positions
* minting options
* transferring collateral
* executing trades

If a failed transaction can later be replayed successfully:

* the user may unknowingly execute an **old intent**
* funds may be used **without the user initiating a new transaction**
* the replay could occur **under different market conditions**

This breaks an important expectation:

```text
A failed transaction should not remain executable indefinitely.
```

### Concrete Walkthrough (Alice & Attacker)

**Setup**

Alice has:

```text
10,000 USDC
```

She signs a meta-transaction to call:

```solidity
operate(actions)
```

which will spend **10,000 USDC**.

Nonce:

```yaml
nonce = 5
```

**Step 1 — Balance Changes**

Before the meta-transaction is executed, Alice transfers:

```text
1,000 USDC → Bob
```

Alice now has:

```text
9,000 USDC
```

**Step 2 — Meta-Transaction Executes**

The relayer submits the transaction.

Inside `executeMetaTransaction`:

1. Nonce increments

```yaml
nonce = 6
```

2. Contract calls:

```solidity
operate(actions)
```

But the operation requires **10,000 USDC**.

Since Alice now has **9,000**, the call fails.

This triggers:

```solidity
require(success)
```

which **reverts the transaction**.

Because of the revert:

```text
nonce increment is undone
nonce returns to 5
```

**Step 3 — Conditions Change Later**

Days later Bob returns the funds:

```text
Bob → Alice = 1,000 USDC
```

Alice now again has:

```text
10,000 USDC
```

**Step 4 — Replay Attack**

An attacker (or anyone holding the signature) resubmits the **same signed meta-transaction**.

The contract checks:

```text
signature valid ✔
nonce == 5 ✔
```

So the transaction executes successfully.

Alice's funds are now used **without her submitting a new transaction**.

> **Analogy**:
> Imagine signing a cheque that fails because your bank balance is temporarily low. If the bank keeps the cheque valid instead of voiding it, someone could cash it later when your balance increases.

## Vulnerable Code Reference

### 1) Nonce Increment Before Execution

```solidity
uint256 currentNonce = _nonces[metaAction.from];

unchecked {
    _nonces[metaAction.from] = currentNonce + 1;
}
```

Nonce is incremented **before the operation executes**.

### 2) Low-Level Call Execution

```solidity
(bool success, bytes memory returnData) = address(this).call(
    abi.encodePacked(
        abi.encodeWithSelector(
            IController(address(this)).operate.selector,
            metaAction.actions
        ),
        metaAction.from
    )
);
```

The contract performs a low-level call to execute the user's action.

### 3) Revert on Failure

```solidity
require(success, "unsuccessful function call");
```

If the call fails, the transaction **reverts completely**, undoing the nonce increment.

This allows the same signature to remain valid and be replayed later.

## Recommended Mitigation

### 1. Ensure nonce increments persist even if execution fails

Avoid reverting after the external call so the nonce cannot roll back.

Example:

```solidity
(bool success, bytes memory returnData) = address(this).call(...);

if (!success) {
    return returnData;
}
```

This ensures:

```text
nonce increments remain permanent
signature cannot be replayed
```

### 2. Follow established implementations

Use patterns similar to **OpenZeppelin MinimalForwarder**, where:

* nonce increments
* execution result is returned
* failures do not revert nonce updates

### 3. Protect against insufficient gas griefing

Require that the relayer provides sufficient gas to complete execution to prevent griefing scenarios.

### 4. Add replay-safety tests

Unit tests should ensure:

* failed meta-transactions cannot be replayed
* nonce increments persist even when calls fail

## Pattern Recognition Notes

### Revert-Induced Nonce Rollback

When nonce updates occur before execution but the transaction can revert afterward, the nonce update may be undone. This creates replay opportunities.

### Meta-Transaction Replay Surface

Systems relying on signatures must ensure that **each signature becomes permanently invalid after a single attempt**, regardless of execution success.

### Time-Dependent Transaction Risks

Transactions relying on dynamic state (balances, timestamps, or market conditions) are especially dangerous if replayable, since they may fail initially but succeed later.

### Failure Handling in Meta-Tx Systems

Meta-transaction frameworks should **separate execution failure from nonce invalidation** to ensure that signatures cannot remain reusable.

### Boundary Enforcement on Signature Usage

Each signature should be considered **consumed once submitted**, not only when execution succeeds.

## Quick Recall (TL;DR)

* **Bug**: Failed meta-transactions revert nonce increment.
* **Impact**: Same signed meta-transaction can be replayed later.
* **Risk**: Unexpected execution of old user intents when conditions change.
* **Fix**: Ensure nonce increments persist even if execution fails and follow safe meta-transaction patterns.
