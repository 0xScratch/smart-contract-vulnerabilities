# Transaction DoS via permit() Front-Running in RouterV2

* **Severity**: Medium
* **Source**: [MixBytes Report](https://github.com/mixbytes/audits_public/blob/master/EYWA/CLP/README.md#5-transaction-dos-via-permit-front-running)
* **Affected Contract**: [`RouterV2.sol`](https://github.com/eywa-protocol/eywa-clp/blob/d68ba027ff19e927d64de123b2b02f15a43f8214/contracts/RouterV2.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Mempool Frontrunning / Permit Replay Protection Bypass

## Summary

`RouterV2` supports a pipeline of cross-chain operations. One of these operations is `PERMIT_CODE`, which calls:

```solidity
IERC20WithPermit(p.token).permit(owner, address(this), amount, deadline, v, r, s);
```

ERC20 `permit()` relies on **signatures**, which become publicly visible once the user's transaction enters the mempool. Anyone can copy these `v, r, s` parameters and call `permit()` **before** the legitimate `start()` call executes.

Because ERC20 signatures are **single-use**, the first execution succeeds — and every subsequent execution with the same arguments **reverts**.

Thus:

* An attacker can **frontrun** the user's router transaction.
* The router's internal `permit()` call **reverts**, halting the entire start() pipeline.
* The legitimate user experiences a full **transaction DoS**, even though the attacker did not gain allowance or funds.

The intended user action simply becomes impossible until they prepare a *new* permit signature.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. The user calls:

   ```solidity
   start(["P", "LM"], [permitParams, lockMintParams])
   ```

2. Router executes `PERMIT_CODE`:

   ```solidity
   token.permit(owner → router)
   ```

3. Then Router continues to lock/mint/burn/wrap/unwrap depending on the pipeline.

This lets users avoid sending separate "approve()" transactions.

### What Actually Happens (Bug)

The permit signature submitted with the transaction is visible in the mempool:

```solidity
(owner, spender, amount, deadline, v, r, s)
```

An attacker does:

```solidity
token.permit(owner, router, amount, deadline, v, r, s)
```

immediately, before the router's `start()` transaction executes.

Effects:

1. **Attacker's permit() succeeds**, granting allowance to the router (harmless but impactful).
2. When RouterV2 processes the operation `PERMIT_CODE`, it tries to call `permit()` again:

   ```solidity
   token.permit(...)
   ```

3. But **ERC20 permit() signature replay protection** causes a **revert** on second use.
4. Since the router wrapped the permit call with **no try/catch**, the revert bubbles up →
   **the entire start() fails**, killing the transaction.

The user's intended bridging flow is DoS'ed.

### Why This Matters

* An attacker doesn't need to steal funds — simply submitting the permit early breaks the pipeline.
* This creates a **denial-of-service against any multi-op start() call that uses PERMIT_CODE**.
* User can be griefed cheaply: attacker only pays gas for a single permit() call.
* The router cannot proceed to locking, minting, or cross-chain forwarding.
* The effect is **permanent** for that signature — user must create a *new* signature.

This is an especially nasty DoS in a multi-chain router context, where multi-opcode execution is expected to be atomic.

## Concrete Walkthrough (Alice & Mallory)

**Scenario**:
Alice wants to bridge `USDC` from Chain A → Chain B.

### Step 1 — Alice prepares a transaction

```solidity
operations = ["P", "LM"]
params = [permitParams, lockMintParams]
```

She signs `permit(owner → router, 500 USDC)`.

Her transaction enters the mempool.

### Step 2 — Mallory watches the mempool

She copies the signature `v, r, s` and frontruns:

```solidity
token.permit(owner, router, 500 USDC, deadline, v, r, s)
```

This succeeds.

### Step 3 — Alice's router execution begins

Inside `_executeOp()`:

```solidity
if (PERMIT_CODE == op) {
    token.permit(owner, router, amount, deadline, v, r, s); // second call
}
```

This now **reverts**, because:

* The signature is single-use.
* The EIP-2612 nonce has already incremented.
* Router V2 does NOT catch the revert.

### Step 4 — Entire start() reverted

Alice's bridging transaction never proceeds to `LM`.

This is a **pure denial-of-service** attack requiring no further effort.

## Vulnerable Code Reference

### 1) Raw permit() call — no try/catch

```solidity
if (PERMIT_CODE == op) {
    PermitParams memory p = abi.decode(params, (PermitParams));
    IERC20WithPermit(p.token).permit(
        p.owner,
        address(this),
        p.amount,
        p.deadline,
        p.v,
        p.r,
        p.s
    );
}
```

If this reverts, `_executeOp()` reverts, and so does the full execution pipeline.

### 2) permit() is not authenticated

Anyone can call:

```solidity
token.permit(owner, router, ...)
```

This is valid ERC20 behavior
→ Router wrongly assumes that only the router will call it.

## Recommended Mitigation

### **1. Surround permit() with try/catch**

Prevents the revert from bubbling up:

```solidity
try IERC20WithPermit(p.token).permit(
    p.owner,
    address(this),
    p.amount,
    p.deadline,
    p.v,
    p.r,
    p.s
) {} catch {
    // permit already used → safe to continue
}
```

This was the recommended and adopted mitigation.

### **2. (Optional) Pre-check current allowance**

Skip permit if the spender already has enough allowance:

```solidity
if (token.allowance(p.owner, address(this)) < p.amount) {
    // only then try permit
}
```

### **3. (Optional) Support replay-safe permit types**

Such as **Permit2**, which uses unlimited nonce spaces.

## Pattern Recognition Notes

* **Permit signatures are single-use** — calling permit() twice will always revert.
* **Mempool visibility makes signatures public** unless protected with private mempools.
* **Bridging pipelines are fragile** — a revert in an early step breaks the entire chain of operations.
* **DoS-by-griefing is common** when protocols assume permit() will always be called by a trusted actor.
* **Atomic pipelines require defensive try/catch** on any external call that may revert.

## Quick Recall (TL;DR)

* **Bug**: Router calls `permit()` directly. Signature becomes public in mempool. Attacker frontruns and uses the permit first.
* **Impact**: Router's permit call reverts → entire start() reverts → user gets DoS'ed.
* **Fix**: Wrap permit() in `try/catch` so the router doesn't fail if signature was pre-used.
