# ERC-1271 Signer Does Not Pin `appData`, Allowing Solver-Controlled CoW Surplus Siphoning

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-chainlink-payment-abstraction-v2/submissions/S-175)
* **Affected Contract**: [GPV2CompatibleAuction.sol](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/GPV2CompatibleAuction.sol), [GPv2Order.sol](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/vendor/@cowprotocol/contracts/src/contracts/libraries/GPv2Order.sol)
* **Vulnerability Type**: Business Logic Flaw / Incomplete Validation / ERC-1271 Authorization Bug

## Summary

The auction integrates with CoW Protocol through ERC-1271 signatures. Instead of producing traditional ECDSA signatures, the auction contract approves orders by implementing:

```solidity
isValidSignature(...)
```

The function validates many important order parameters such as:

* Sell token
* Buy token
* Sell amount
* Buy amount (auction floor)
* Receiver
* Expiry
* Order type

However, it never validates:

```solidity
order.appData
```

This is dangerous because, in CoW Protocol, `appData` is not merely informational metadata. It can contain execution-related information such as partner-fee configuration and surplus distribution instructions.

As a result, a malicious or colluding CoW solver can submit an otherwise valid order containing attacker-controlled `appData`, causing a portion of the order's positive execution surplus to be redirected away from Chainlink while still satisfying all auction pricing requirements.

The auction receives at least its minimum acceptable amount of LINK, but additional value generated during execution can be siphoned elsewhere.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The auction wants to sell assets through CoW Protocol.

Example:

```text
Auction sells:
100,000 USDC

Auction requires:
At least 5,250 LINK
```

The auction contract acts as the signer and approves orders through ERC-1271:

```text
CoW Settlement
      ↓
isValidSignature()
      ↓
Auction approves order
```

The goal is simple:

```text
Never sell below auction price.
```

### What Actually Happens (Bug)

The auction validates:

```solidity
order.sellAmount
order.buyAmount
order.receiver
order.kind
order.feeAmount
```

but never validates:

```solidity
order.appData
```

This means an attacker-controlled solver can freely choose any `appData` value.

The contract only verifies that:

```solidity
hash == GPv2Order.hash(order, domainSeparator)
```

which merely proves:

```text
Order matches hash
```

It does **not** prove:

```text
Order was authorized by the protocol
```

for a particular `appData`.

As long as the attacker recomputes the hash using their modified `appData`, the signature remains valid.

### Why This Matters

Many developers assume:

```text
appData = metadata
```

However, within CoW Protocol, `appData` can influence how value generated during settlement is distributed.

Most importantly:

```text
Partner Fees
Surplus Distribution
Referral Logic
```

can be driven through information encoded within `appData`.

CoW's own documentation specifically warns custom ERC-1271 signers against allowing arbitrary `appData`.

### Understanding Surplus

The auction only enforces a minimum acceptable execution price.

Suppose:

```text
Auction Minimum:
5,250 LINK
```

But market conditions allow a better execution:

```text
Actual Execution:
5,350 LINK
```

Difference:

```text
100 LINK
```

This extra value is called:

```text
Surplus
```

### What Should Happen

```text
Minimum Required:
5,250 LINK

Actual Execution:
5,350 LINK

Surplus:
100 LINK
```

The entire benefit should accrue to Chainlink.

```text
Chainlink receives:
5,350 LINK
```

### What Actually Happens

A malicious solver submits:

```text
sellAmount = 100,000 USDC
buyAmount  = 5,250 LINK
```

All auction validations pass.

The solver then chooses attacker-controlled:

```text
appData
```

containing partner-fee instructions.

Settlement executes at:

```text
5,350 LINK
```

Instead of all surplus reaching Chainlink:

```text
100 LINK surplus
```

part of it gets redirected.

Example:

```text
53.5 LINK → Attacker
46.5 LINK → Chainlink
```

while Chainlink still receives:

```text
5,296.5 LINK
```

which remains above the auction's minimum floor.

All validations pass successfully.

### Concrete Walkthrough (Alice & Malicious Solver)

**Setup**

Auction parameters:

```text
100,000 USDC
Minimum Buy:
5,250 LINK
```

Auction is live.

**Step 1 — Solver Creates Order**

The solver builds:

```text
sellAmount = 100,000 USDC
buyAmount  = 5,250 LINK
receiver   = Auction
feeAmount  = 0
```

All valid.

**Step 2 — Solver Modifies appData**

Instead of:

```text
appData = 0x0
```

the solver sets:

```text
appData = attacker-controlled value
```

that encodes surplus-sharing instructions.

**Step 3 — Solver Recomputes Hash**

The hash includes `appData`:

```solidity
hash(order)
```

Therefore the solver simply computes:

```text
new appData
↓
new hash
```

The self-consistency check still succeeds.

**Step 4 — Auction Approves Order**

`isValidSignature()` verifies:

```text
Sell amount ✓
Buy amount ✓
Receiver ✓
Expiry ✓
Fee amount ✓
```

but never verifies:

```text
appData
```

The order is approved.

**Step 5 — Settlement Executes**

Market execution achieves:

```text
5,350 LINK
```

instead of:

```text
5,250 LINK
```

creating:

```text
100 LINK surplus
```

The malicious `appData` redirects part of this value.

Result:

```text
Chainlink receives less surplus
Attacker receives siphoned value
```

without violating any auction pricing rule.

> **Analogy:** Imagine selling a house with a contract that says "I must receive at least $500,000." The buyer later finds someone willing to pay $520,000. Normally you would receive the extra $20,000. Instead, a hidden clause inserted into the paperwork sends part of that bonus to a middleman. You still receive more than $500,000, so the sale succeeds, but value that should have belonged to you is diverted elsewhere.

## Vulnerable Code Reference

### 1) Hash Check Only Verifies Self-Consistency

```solidity
if (hash != GPv2Order.hash(order, i_gpV2Settlement.domainSeparator())) {
    revert InvalidOrderId(hash);
}
```

This verifies:

```text
Order matches supplied hash
```

but does not verify:

```text
appData matches protocol expectations
```

### 2) Economic Fields Are Checked

```solidity
if (order.buyAmount < minBuyAmount) {
    revert InsufficientBuyAmount(...);
}
```

The auction enforces:

```text
Minimum execution price
```

correctly.

### 3) feeAmount Is Checked

```solidity
if (order.feeAmount > 0) {
    revert InvalidFeeAmount();
}
```

This prevents explicit order fees.

However:

```text
Partner-fee behavior
```

can still be influenced through `appData`.

### 4) appData Is Never Validated

No code exists similar to:

```solidity
if (order.appData != expectedAppData) {
    revert InvalidAppData();
}
```

Therefore:

```text
Any appData value is accepted.
```

## Recommended Mitigation

### 1. Pin appData to a Fixed Value (Recommended)

If the auction does not require custom appData:

```solidity
if (order.appData != bytes32(0)) {
    revert InvalidAppData();
}
```

This completely removes the attack surface.

### 2. Store an Explicit appData Commitment

If non-zero appData is required:

```solidity
mapping(bytes32 => bool) allowedAppData;
```

or

```solidity
bytes32 immutable allowedAppData;
```

and validate:

```solidity
if (order.appData != allowedAppData) {
    revert InvalidAppData();
}
```

### 3. Treat appData as an Economic Parameter

When integrating with CoW Protocol:

```text
sellAmount
buyAmount
feeAmount
appData
```

should all be considered economically meaningful fields.

Any field capable of influencing settlement behavior should be explicitly authorized.

### 4. Add Regression Tests

Add tests proving:

* Non-approved `appData` values revert.
* Modified `appData` cannot be used even when all other order parameters remain valid.
* Surplus-routing metadata cannot be altered by external actors.

## Pattern Recognition Notes

### Self-Consistency ≠ Authorization

A very common audit mistake is confusing:

```text
Hash matches data
```

with:

```text
Data is authorized.
```

The hash check only proves:

```text
Attacker submitted matching inputs.
```

It does not prove:

```text
Protocol approved those inputs.
```

### Validate Every Economically Meaningful Field

Auditors often focus on:

```text
Amounts
Prices
Recipients
```

and overlook auxiliary fields.

When integrating external protocols, always ask:

> Can this field influence money flow?

If yes:

```text
It must be validated.
```

### ERC-1271 Signers Define Their Own Trust Boundary

Traditional signatures authorize an exact message.

ERC-1271 contracts effectively implement:

```text
Custom Authorization Logic
```

Every field not explicitly validated becomes attacker-controlled.

### Surplus Is Value Too

Protocols frequently validate:

```text
Minimum Amount Out
```

but forget to reason about:

```text
Who receives value above the minimum?
```

Positive slippage and surplus often become attack surfaces when integrations introduce additional routing logic.

### External Protocol Metadata Can Be Dangerous

Fields that appear harmless:

```text
metadata
memo
extraData
appData
```

sometimes carry execution instructions.

Always verify how external protocols interpret those fields before assuming they are informational only.

## Quick Recall (TL;DR)

* **Bug**: `isValidSignature()` never validates `order.appData`.
* **Root Cause**: Hash verification only checks self-consistency, not whether `appData` is authorized.
* **Impact**: Malicious or colluding CoW solvers can inject surplus-routing metadata and siphon part of positive execution surplus.
* **Why Checks Don't Stop It**: The auction only enforces a minimum buy amount; surplus above that floor remains exploitable.
* **Fix**: Pin `appData` to a known value (ideally `bytes32(0)`) or compare it against an explicit on-chain commitment.
* **Key Lesson**: In ERC-1271 integrations, every economically meaningful field must be explicitly authorized—even fields that appear to be metadata.
