# Arbitrary External Call → Token Drain via User Allowances (GenericBridgeFacet)

* **Severity**: High
* **Source**: [SpearBit (Solodit)](https://solodit.cyfrin.io/issues/too-generic-calls-in-genericbridgefacet-allow-stealing-of-tokens-spearbit-lifi-pdf)
* **Affected Contract**: `GenericBridgeFacet.sol`, `LibSwap.sol`
* **Vulnerability Type**: Arbitrary External Call / Access Control Failure / Allowance Abuse

## Summary

`GenericBridgeFacet` allowed users to supply arbitrary `callTo` addresses and arbitrary `callData`, which were executed via low-level `.call()` inside both `swapAndStartBridgeTokensGeneric()` and `_startBridge()`.

Because users had previously granted large token allowances to the LiFi Diamond contract, attackers could craft malicious calldata that invoked `transferFrom()` on ERC20 tokens. Since the Diamond contract was an approved spender, the transfer succeeded — allowing attackers to drain tokens from victims.

The issue was aggravated by the absence of:

* Whitelisting of allowed target contracts
* Validation of allowed function selectors
* Proper access control for internal execution paths

The feature was later removed entirely from deployments.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The design goal was flexibility:

1. User submits swap + bridge instructions.
2. Contract performs:

   ```solidity
   callTo.call(callData)
   ```

3. This allows integration with many DEXs and bridges dynamically.
4. After swapping, bridging starts.

In theory, this makes the system modular and extensible.

### What Actually Happens (Bug)

The contract does:

```solidity
(bool success, bytes memory res) = _swapData.callTo.call{ value: nativeValue }(_swapData.callData);
```

There is **no restriction** on:

* `_swapData.callTo`
* `_swapData.callData`

So anyone can make the contract call **any function on any contract**.

That includes:

```solidity
USDC.transferFrom(victim, attacker, amount);
```

If the victim previously did:

```solidity
USDC.approve(LiFiDiamond, 1_000_000);
```

Then:

* `msg.sender` inside USDC = LiFi Diamond
* Diamond is approved spender
* Transfer succeeds

💥 Tokens drained.

## Concrete Walkthrough (Alice & Mallory)

### Setup

* Alice previously used LI.FI
* She approved 1,000,000 USDC to the LiFi Diamond contract.

### Mallory's Attack

Mallory crafts malicious input:

```plaintext
callTo = USDC contract address
callData = transferFrom(Alice, Mallory, 500_000)
```

Mallory calls:

```solidity
swapAndStartBridgeTokensGeneric(...)
```

Inside:

```
LibSwap.swap(...)
   → arbitrary .call()
```

The Diamond executes:

```
USDC.transferFrom(Alice, Mallory, 500_000)
```

Because:

* Diamond is approved spender
* USDC sees valid allowance

Transfer succeeds.

Alice loses funds.

### Why This Is Severe

This is not just "bad validation".

It effectively turns the contract into:

> A universal execution engine funded by user allowances.

Anyone can instruct it to:

* Transfer tokens
* Call other protocols
* Even call itself

## Additional Exploitable Behaviors

### 1️⃣ Self-Call Reentrancy

Because `callTo` is arbitrary, attacker could set:

```plaintext
callTo = address(this)
```

If certain internal functions lacked `nonReentrant`, this could:

* Enable reentrancy
* Bypass execution guards

---

### 2️⃣ Cancellation of Other Users' Transfers

If a function like:

```solidity
cancelTransfer(transactionId)
```

lacked proper sender validation, attacker could force the contract to call it.

### 3️⃣ Bypass of `msg.sender == address(this)` Protections

Some functions rely on:

```solidity
require(msg.sender == address(this));
```

Normally safe.

But if the Diamond calls itself via `.call()`:

* `msg.sender` becomes `address(this)`
* Check passes
* Protected logic executes

## Vulnerable Code Reference

### 1) Arbitrary Call in `LibSwap`

```solidity
library LibSwap {
    function swap(bytes32 transactionId, SwapData calldata _swapData) internal {
        (bool success, bytes memory res) =
            _swapData.callTo.call{ value: nativeValue }(_swapData.callData);
    }
}
```

### 2) Arbitrary Call in `_startBridge`

```solidity
function _startBridge(BridgeData memory _bridgeData) internal {
    (bool success, bytes memory res) =
        _bridgeData.callTo.call{ value: value }(_bridgeData.callData);
}
```

## Root Cause

Unrestricted low-level `.call()` using:

* User-controlled target address
* User-controlled calldata
* No whitelist
* No function selector validation
* Contract holding token allowances

This is a classic **Arbitrary External Call + Allowance Abuse** vulnerability.

## Recommended Mitigation

### 1️⃣ Whitelist Allowed Contracts

Only allow calls to:

* Approved DEX contracts
* Approved bridge contracts

### 2️⃣ Whitelist Allowed Function Selectors

Validate:

```solidity
bytes4 selector = bytes4(callData[:4]);
require(allowedSelectors[selector]);
```

### 3️⃣ Avoid Generic `.call()` Execution Engines

If flexibility is required:

* Build specific adapter contracts
* Hardcode integration paths

### 4️⃣ Remove Feature (Final Resolution)

LiFi removed the feature entirely:

* `GenericBridgeFacet` deleted
* Not re-enabled in deployments

This eliminated the arbitrary call surface.

## Pattern Recognition Notes

### 🔥 1. Arbitrary Call + Allowance = Critical Risk

If a contract:

* Has ERC20 allowances
* And allows user-controlled `.call()`

You almost always have a token drain vector.

### 🔥 2. Diamond Architecture Amplifies Risk

Because the Diamond:

* Centralizes allowances
* Routes execution across facets

A single unsafe facet exposes the entire system.

### 🔥 3. msg.sender Confusion Attacks

If you see:

```solidity
require(msg.sender == address(this))
```

Always check:
Can the contract call itself via `.call()`?

### 🔥 4. Flexibility vs Security Tradeoff

"Generic execution engines" are attractive,
but without strict validation they become:

> Remote-controlled exploit surfaces.

## Quick Recall (TL;DR)

* **Bug**: Unrestricted `.call()` using user-controlled address + calldata
* **Impact**: Attackers used existing token allowances to call `transferFrom()` and drain user funds
* **Additional Risks**: Self-call bypass, reentrancy, cancellation abuse
* **Fix**: Feature removed; GenericBridgeFacet deleted
