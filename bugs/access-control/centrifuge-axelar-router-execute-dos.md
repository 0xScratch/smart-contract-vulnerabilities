# Cross‑Chain Execute DoS via Over‑Restrictive Origin Check in Axelar Router

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-09-centrifuge-findings/issues/537)
* **Affected Contract**: [Router.sol](https://github.com/code-423n4/2023-09-centrifuge/blob/512e7a71ebd9ae76384f837204216f26380c9f91/src/gateway/routers/axelar/Router.sol#L44)
* **Vulnerability Type**: Denial of Service (DoS) / Access Control / Cross‑Chain Messaging

## Summary

The `Router.execute()` function in Centrifuge's Axelar Router is protected by the `onlyCentrifugeChainOrigin` modifier, which enforces that `msg.sender` must be the `axelarGateway` contract. However, the Axelar Gateway contract **never directly calls** `execute()`.

Because of this incorrect assumption, the `execute()` function becomes **un-callable**, preventing cross‑chain messages from being processed. As a result, **all cross‑chain commands fail**, effectively breaking core protocol functionality.

This creates a **Denial of Service (DoS)** condition where cross‑chain communication becomes permanently unavailable.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The router is designed to process cross‑chain messages coming from Axelar.

The expected flow:

1. A message is sent from another chain
2. Axelar validates the message
3. Router `execute()` is called
4. Router processes the command

To secure this flow, the router validates:

* The message is approved by Axelar
* The message comes from the correct chain
* The message comes from the correct address

This validation is implemented using:

* `axelarGateway.validateContractCall()`
* `sourceChain` validation
* `sourceAddress` validation

### What Actually Happens (Bug)

The router adds an additional restriction:

```solidity
require(msg.sender == address(axelarGateway), "AxelarRouter/invalid-origin");
```

This means:

> Only the Axelar Gateway contract can call `execute()`

However, the **Axelar Gateway contract never calls `execute()`**.

Therefore:

* `msg.sender` will never equal `axelarGateway`
* The condition always fails
* `execute()` always reverts

As a result, **cross‑chain commands are never executed**.

### Why This Matters

* Cross‑chain messaging stops working
* Commands from other chains never execute
* Protocol features relying on cross‑chain messaging break
* System functionality becomes partially unusable

This effectively results in a **protocol‑level DoS**.

## Concrete Walkthrough (Alice & Bob)

### Setup

* Centrifuge uses Axelar for cross‑chain messaging
* Router `execute()` processes incoming messages

### Step 1 — Alice sends cross‑chain command

Alice sends a command from another chain.

### Step 2 — Axelar validates message

Axelar confirms:

* Message is valid
* Sender is legitimate

### Step 3 — Router `execute()` should run

The router attempts to process the command.

### Step 4 — Modifier blocks execution

The modifier checks:

```solidity
require(msg.sender == address(axelarGateway));
```

But `msg.sender` is **not** `axelarGateway`.

Execution reverts.

### Result

* Alice's command fails
* All cross‑chain messages fail
* Protocol functionality breaks

### Analogy

Imagine a package delivery system:

* Packages are verified at the warehouse
* Courier delivers packages
* Office only accepts packages from the **warehouse manager**

But:

* Manager never delivers packages
* Courier always delivers

Result:

No packages are accepted → System breaks

## Vulnerable Code Reference

### 1) Over‑Restrictive Origin Validation

```solidity
modifier onlyCentrifugeChainOrigin(
    string calldata sourceChain,
    string calldata sourceAddress
) {
    require(msg.sender == address(axelarGateway), "AxelarRouter/invalid-origin");
    require(
        keccak256(bytes(axelarCentrifugeChainId)) == keccak256(bytes(sourceChain)),
        "AxelarRouter/invalid-source-chain"
    );
    require(
        keccak256(bytes(axelarCentrifugeChainAddress)) == keccak256(bytes(sourceAddress)),
        "AxelarRouter/invalid-source-address"
    );
    _;
}
```

### 2) Execute Function Becomes Unreachable

Because `msg.sender` is never `axelarGateway`, `execute()` cannot run.

## Root Cause

Incorrect assumption that `axelarGateway` directly calls `execute()`.

In reality:

* Axelar validates messages
* But does not directly call router `execute()`

This creates a permanently failing condition.

## Impact

* Cross‑chain commands fail
* Protocol functionality breaks
* Cross‑chain operations unavailable
* Protocol enters partial DoS state

Impact Severity depends on how critical cross‑chain functionality is.

## Recommended Mitigation

### Primary Fix — Remove msg.sender Restriction

```solidity
modifier onlyCentrifugeChainOrigin(
    string calldata sourceChain,
    string calldata sourceAddress
) {
    require(
        keccak256(bytes(axelarCentrifugeChainId)) == keccak256(bytes(sourceChain)),
        "AxelarRouter/invalid-source-chain"
    );
    require(
        keccak256(bytes(axelarCentrifugeChainAddress)) == keccak256(bytes(sourceAddress)),
        "AxelarRouter/invalid-source-address"
    );
    _;
}
```

### Alternative Defensive Fix

If additional validation is required:

* Rely on `validateContractCall()`
* Validate source chain
* Validate source address

These checks already ensure legitimacy.

## Pattern Recognition Notes

### Over‑Restrictive Access Control

Adding unnecessary access restrictions can unintentionally block legitimate execution paths.

### Incorrect External Protocol Assumptions

Assuming external protocol behavior without verification can introduce critical bugs.

### Cross‑Chain DoS Risks

Cross‑chain routers must ensure message execution paths remain reachable.

### Redundant Security Checks

Duplicate validations increase complexity and risk of misconfiguration.

### Integration Risk Pattern

Cross‑protocol integrations are particularly vulnerable to:

* Wrong assumptions
* Incorrect call flows
* Access control mistakes

## Quick Recall (TL;DR)

* **Bug**: `msg.sender == axelarGateway` restriction
* **Issue**: AxelarGateway never calls `execute()`
* **Impact**: `execute()` becomes unreachable
* **Result**: Cross‑chain DoS
* **Fix**: Remove `msg.sender` restriction
