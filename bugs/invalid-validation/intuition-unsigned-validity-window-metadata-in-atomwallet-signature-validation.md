# Unsigned Validity Window Metadata in AtomWallet Signature Validation

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-intuition/submissions/F-9)
* **Affected Contract**: [AtomWallet.sol](https://github.com/code-423n4/2026-03-intuition/blob/314b7d4d9ccbaf27e4484a6c0706af83d3f75f36/src/protocol/wallet/AtomWallet.sol#L285)
* **Vulnerability Type**: Signature Validation / Unsigned Metadata / Authorization Bypass

## Summary

`AtomWallet` supports ERC-4337 signature validity windows by allowing `validUntil` and `validAfter` timestamps to be appended to the signature bytes. However, these timestamps are extracted from the signature payload **without being included in the signed message hash**.

As a result, the wallet verifies ownership only over `userOpHash`, while blindly trusting externally supplied validity metadata. Any relayer or observer can modify the appended `validUntil` / `validAfter` values while reusing the same valid ECDSA signature.

This breaks the intended time-restriction security model and allows attackers to arbitrarily extend, remove, or alter the execution window of signed UserOperations.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol wants users to sign operations with optional time restrictions.

Supported signature formats:

#### Standard Signature

```text
[r || s || v]
```

Meaning:

> "I approve this UserOperation."

#### Time-Restricted Signature

```text
[r || s || v || validUntil || validAfter]
```

Meaning:

> "I approve this UserOperation ONLY during this time window."

Example:

```text
validAfter  = now
validUntil  = now + 1 minute
```

Meaning:

> "This operation expires after 1 minute."

This is especially useful in ERC-4337 systems where signed operations may sit in relayer queues or mempools before execution.

## What Actually Happens (Bug)

Although `validUntil` and `validAfter` are appended to the signature bytes, they are **not included in the actual cryptographic message being signed**.

The wallet computes the recovered signer using only:

```solidity
keccak256(
    abi.encodePacked(
        "\x19Ethereum Signed Message:\n32",
        userOpHash
    )
)
```

The validity window metadata is extracted separately afterward:

```solidity
_extractValidUntilAndValidAfterFromSignature(userOp.signature);
```

This creates a dangerous mismatch:

* The contract **trusts** `validUntil` and `validAfter`
* But the signer never actually authenticated them

As a result, anyone can modify these values without invalidating the signature.

## Why This Matters

The protocol intended users to control BOTH:

```text
1. WHAT operation executes
2. WHEN it may execute
```

But due to the bug, users only controlled:

```text
1. WHAT operation executes
```

while attackers controlled:

```text
2. WHEN it may execute
```

This completely defeats the purpose of expiration windows.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Alice signs a UserOperation:

```text
Swap 100 USDC
```

with:

```text
validUntil = now + 1 minute
```

She expects:

> "If this operation does not execute within 1 minute, it becomes unusable forever."

### What Alice THINKS She Signed

```text
(
    userOpHash,
    validUntil = 1 minute
)
```

### What She ACTUALLY Signed

Only:

```text
userOpHash
```

The timestamp metadata is merely appended afterward.

### Mallory Observes The Signature

Since ERC-4337 UserOperations are visible to bundlers/relayers:

* Mallory can see:

  * `userOp`
  * ECDSA signature
  * appended timestamps

### Original Window Expires

After 1 minute:

```text
validUntil < block.timestamp
```

The operation SHOULD be permanently unusable.

### Mallory Modifies Unsigned Metadata

Mallory keeps the SAME valid ECDSA signature:

```text
[r,s,v]
```

but changes:

```text
validUntil = now + 30 days
```

### Why Validation Still Passes

Because signer recovery still uses only:

```text
userOpHash
```

which has not changed.

The wallet therefore recovers Alice's address successfully and accepts the operation.

### Final Result

Alice intended:

```text
"usable for 1 minute"
```

but Mallory transformed it into:

```text
"usable for 30 days"
```

without forging any signature.

> **Analogy**: Imagine signing a cheque that says:
>
> ```text
> "Valid until tomorrow"
> ```
>
> but the expiration date is written outside the signed region. Someone later erases:
>
> ```text
> tomorrow
> ```
>
> and replaces it with:
>
> ```text
> next year
> ```
>
> The signature still verifies because the expiration text was never part of what was signed.

## Vulnerable Code Reference

### 1) Validity Window Extracted Separately

```solidity
(uint48 validUntil, uint48 validAfter, bytes memory signature) =
    _extractValidUntilAndValidAfterFromSignature(userOp.signature);
```

The timestamps are parsed from trailing bytes but are not authenticated.

### 2) Signature Hash Omits Validity Metadata

```solidity
bytes32 hash =
    keccak256(
        abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            userOpHash
        )
    );
```

Only `userOpHash` is signed.

Missing:

```solidity
validUntil
validAfter
```

### 3) Signature Validation Trusts Unsigned Metadata

```solidity
bool sigFailed = recovered != owner();
return _packValidationData(sigFailed, validUntil, validAfter);
```

The contract uses attacker-modifiable timestamps as trusted authorization data.

## Root Cause

The protocol appended authorization-critical metadata (`validUntil` and `validAfter`) to the signature payload but failed to include those values inside the signed digest.

This resulted in:

```text
Unsigned security-sensitive metadata
```

The wallet therefore treated attacker-controlled data as authenticated signer intent.

## Recommended Mitigation

### 1) Cryptographically Bind Validity Metadata To The Signature

Include `validUntil` and `validAfter` in the signed payload.

Example fix:

```solidity
bytes32 signedHash = userOpHash;

if (userOp.signature.length == 77) {
    signedHash = keccak256(
        abi.encodePacked(
            userOpHash,
            validUntil,
            validAfter
        )
    );
}

bytes32 hash =
    keccak256(
        abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            signedHash
        )
    );
```

Now any modification to:

```text
validUntil
validAfter
```

changes the signed digest and invalidates the signature.

### 2) Treat Metadata As Untrusted Unless Signed

Any data affecting:

* authorization
* replay protection
* permissions
* expiry
* execution conditions

must always be cryptographically bound to signer intent.

### 3) Add Signature Integrity Tests

Unit tests should verify:

* modifying `validUntil` invalidates signature
* modifying `validAfter` invalidates signature
* replay attempts outside original window fail

## Pattern Recognition Notes

* **Unsigned Metadata Vulnerability**: Appending data to a signature payload does NOT mean the signer authenticated it. Only values included in the signed hash are protected.
* **Authorization Context Must Be Signed**: Expiry windows, nonces, permissions, deadlines, chain IDs, and replay protections must always be part of the signed digest.
* **ERC-4337 Relayer Threat Model**: Bundlers and relayers can observe and modify UserOperations before submission. Any unsigned field should be considered attacker-controlled.
* **Cryptographic Boundary Confusion**: Developers often confuse:

  ```text
  "attached to signature"
  ```

  with:

  ```text
  "covered by signature"
  ```

  These are completely different concepts.
* **Replay & Delayed Execution Risk**: Expiry windows are often relied upon to limit stale or dangerous future execution. If attackers can alter those windows, users lose temporal control over authorization.

## Quick Recall (TL;DR)

* **Bug**: `validUntil` and `validAfter` were appended to signature bytes but NOT included in the signed hash.
* **Impact**: Attackers could modify signature validity windows without invalidating the ECDSA signature.
* **Result**: Expired UserOperations could be revived and executed outside the signer's intended timeframe.
* **Fix**: Include validity metadata inside the signed digest before signature recovery.
