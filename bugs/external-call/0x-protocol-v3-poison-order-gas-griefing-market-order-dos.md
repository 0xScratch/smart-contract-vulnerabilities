# Poison Order Gas Griefing Causing Market Order DoS in 0x v3 Exchange

* **Severity**: Medium
* **Source**: [Consensys (Solodit)](https://solodit.cyfrin.io/issues/poison-order-that-consumes-gas-can-block-market-trades-wont-fix-consensys-0x-v3-exchange-markdown)
* **Affected Contract**: `MixinWrapperFunctions.sol`, `MixinSignatureValidator.sol`
* **Vulnerability Type**: Denial of Service (DoS) / Gas Griefing / External Call Abuse

## Summary

The **0x v3 Exchange** supports market order functions that sequentially attempt to fill orders from an order list until the desired trade amount is satisfied. To make the system robust, the implementation uses `_fillOrderNoThrow()` which attempts to fill orders while ignoring failures.

However, during order filling the exchange may perform **external signature validation** through user-provided contracts (e.g., EIP-1271 wallets or validator contracts). These external calls **forward all available gas** to the verifying contract.

A malicious maker can exploit this by creating a **"poison order"** that uses a signature validator contract intentionally designed to **consume all available gas**. When a market order attempts to fill this order, the validator call exhausts the transaction gas, causing the entire transaction to revert before `_fillOrderNoThrow()` can gracefully handle the failure.

Because orderbooks typically prioritize **best-priced orders**, the poisoned order will likely be attempted first, effectively **blocking all market trades**. The attack costs the attacker nothing and can be used to temporarily disable competing exchanges or relayers.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Market orders in the 0x Exchange work roughly like this:

1. A taker submits a **market buy or sell**.
2. The contract receives a **list of candidate orders** sorted by best price.
3. The system attempts to fill them sequentially using `_fillOrderNoThrow()`:

   * If an order succeeds → tokens are transferred.
   * If an order fails → the failure is ignored and the next order is attempted.

4. The process continues until the desired amount is filled.

This design ensures a single bad order cannot prevent the overall trade from completing.

### What Actually Happens (Bug)

The vulnerability occurs because **external signature validation calls forward unlimited gas**.

When the exchange validates certain signature types (e.g., wallet contracts or validators), it performs a `staticcall` to the verifying contract. Since the call forwards all available gas, a malicious validator contract can deliberately consume it.

If the validator contract exhausts the gas:

* the transaction runs out of gas,
* the entire transaction **reverts**, and
* `_fillOrderNoThrow()` cannot catch or ignore the failure.

Thus, a malicious order can **force every market order transaction to revert**.

### Why This Matters

* Attackers can create a **very cheap order** that appears attractive in orderbooks.
* Market trades will always try this order first.
* When the poisoned order is processed, the validator contract consumes all gas.
* The entire transaction reverts before the exchange can skip the order.

This results in a **temporary global trading freeze for market orders**.

The attack is extremely attractive because:

* The attacker pays **no gas unless someone tries to fill the order**.
* The order itself is valid and easily distributed via off-chain orderbooks.
* Competing relayers or exchanges could use this to sabotage each other.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Orderbook for ETH → DAI trades:

| Order | Maker   | Price          |
| ----- | ------- | -------------- |
| 1     | Mallory | **Very cheap** |
| 2     | Alice   | Normal price   |
| 3     | Bob     | Normal price   |

Because Mallory offers the best price, her order appears **first** in the list.

### Mallory Attack

Mallory creates an order with:

* A **validator-based signature**
* A validator contract designed to **consume all gas**

Example malicious validator logic:

```solidity
function isValidSignature(...) external returns (bytes4) {
    while(true) {}
}
```

This contract simply burns all gas.

### Alice Executes Market Buy

Alice calls a market buy function.

The exchange attempts to fill orders sequentially.

```text
Attempt fill order #1 (Mallory)
```

### Signature Validation

The exchange performs:

```solidity
staticcall(verifyingContract)
```

Since **all gas is forwarded**, Mallory's validator contract consumes it.

### Result

The transaction runs out of gas and **reverts entirely**.

Because the transaction reverted:

* `_fillOrderNoThrow()` never gets the chance to ignore the failure.
* The exchange never attempts orders #2 or #3.

Every market trade hitting this order will fail.

> **Analogy**: Imagine a checkout line where the first customer intentionally jams the payment terminal in an infinite loop. Since every shopper must go through that customer first, the entire line stops moving even though all other customers are ready to pay.

## Vulnerable Code Reference

### 1) `_fillOrderNoThrow` forwards all gas via delegatecall

```solidity
bytes memory fillOrderCalldata = abi.encodeWithSelector(
    IExchangeCore(address(0)).fillOrder.selector,
    order,
    takerAssetFillAmount,
    signature
);

(bool didSucceed, bytes memory returnData) =
    address(this).delegatecall(fillOrderCalldata);
```

This call forwards **all remaining gas**, making the call vulnerable to gas exhaustion.

### 2) External signature validation forwards unlimited gas

```solidity
(bool didSucceed, bytes memory returnData) =
    verifyingContractAddress.staticcall(callData);
```

Since the validator contract receives all available gas, it can intentionally exhaust it.

## Recommended Mitigation

### 1. Limit Gas Forwarded to Signature Validators (Primary Fix)

When performing external signature validation, constrain the gas forwarded.

Example:

```solidity
(bool success, bytes memory data) =
    verifyingContractAddress.staticcall{gas: GAS_LIMIT}(callData);
```

This ensures malicious validators cannot consume the entire transaction gas.

### 2. Allow Gas Limits as Order Parameters

Instead of hardcoding a value, the gas limit could be provided:

* as part of the order signature, or
* as a parameter supplied by the taker.

This preserves flexibility while preventing gas exhaustion.

### 3. Defensive Orderbook Filtering

Off-chain relayers and frontends can simulate order fills using:

```yaml
eth_call
```

before submitting transactions to filter obviously malicious orders.

## Pattern Recognition Notes

### Gas Griefing via External Calls

Any system that calls **user-supplied contracts** with unrestricted gas is vulnerable to griefing attacks where the callee deliberately consumes all gas.

### Untrusted Signature Validators

Allowing external contracts to validate signatures introduces a **new trust boundary**. Without constraints, these contracts can disrupt protocol execution.

### Failure-Tolerant Execution vs Gas Exhaustion

Mechanisms like `_fillOrderNoThrow()` rely on catching failures. However, **out-of-gas errors cannot be safely handled**, so failure-tolerant designs must still enforce gas limits.

### Orderbook Poisoning

Off-chain orderbooks typically sort by best price. Attackers can exploit this by advertising malicious orders that are **economically attractive but operationally harmful**.

### External Call Resource Control

Whenever interacting with untrusted contracts, protocols should enforce limits on:

* gas forwarded
* call depth
* execution time

to prevent resource exhaustion attacks.

## Quick Recall (TL;DR)

* **Bug**: External signature validation forwards unlimited gas.
* **Attack**: Malicious validator contract consumes all gas during order validation.
* **Impact**: Entire market order transaction reverts → **trading DoS**.
* **Fix**: Limit gas forwarded to validator contracts and defensively simulate orders before execution.
