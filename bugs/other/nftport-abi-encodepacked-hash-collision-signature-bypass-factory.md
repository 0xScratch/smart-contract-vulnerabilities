# Signature Bypass via `abi.encodePacked` Hash Collision in Factory

* **Severity**: Medium  
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-10-nftport-judging/issues/118)
* **Affected Contract**: [`Factory.sol`](https://github.com/sherlock-audit/2022-10-nftport/blob/main/evm-minting-master/contracts/Factory.sol)  
* **Vulnerability Type**: Authentication Bypass / Hash Collision / Input Encoding

## Summary

The `Factory` contract uses `keccak256(abi.encodePacked(...))` inside the `signedOnly` modifier to verify off-chain signatures. However, multiple user-controlled **dynamic types** (e.g., `string`, `bytes`) are packed together.

Because `abi.encodePacked` does **not include length delimiters**, different input combinations can produce the **same packed byte stream**, leading to hash collisions. An attacker can exploit this to submit parameters that were **never actually signed**, thereby bypassing the intended signature authorization.

This breaks the core security assumption that only signer-approved payloads can execute sensitive operations like deploy and call.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Backend signer approves a specific payload off-chain.
2. User submits the payload + signature.
3. Contract recomputes the hash and verifies the signer.
4. Only the **exact signed parameters** should be accepted.

### What Actually Happens (Bug)

The contract builds the signed message like this:

```solidity
abi.encodePacked(msg.sender, templateName, initdata)
```

Both:

- `templateName` → dynamic (`string`)
- `initdata` → dynamic (`bytes`)

When multiple dynamic types are tightly packed, their boundaries are ambiguous.

From Solidity docs:

```solidity
abi.encodePacked("a", "bc") == abi.encodePacked("ab", "c")
```

So different inputs → same packed bytes → same hash → same valid signature.

### Why This Matters

Because the contract verifies only the **hash**, not the original structured data:

```solidity
address messageSigner = ECDSA.recover(
    ECDSA.toEthSignedMessageHash(message),
    signature
);
```

If two different inputs produce the same `message`, the signature becomes reusable for malicious parameters.

### Concrete Walkthrough (Alice & Mallory)

**Setup**

Backend signs deployment for Alice:

```solidity
msg.sender = Alice
templateName = "ABC"
initdata = 0x1234
```

Signed hash corresponds to packed bytes:

```text
[Alice][ABC][1234]
```

**Mallory attack**

Mallory crafts different inputs:

```solidity
templateName = "AB"
initdata = "C1234"
```

Packed result:

```text
[Alice][AB][C1234]
```

Because there are no boundaries, the byte stream matches the signed one.

**Result**

* Signature check passes ✅  
* But parameters differ ❌  
* Unauthorized deployment/call executes 🚨  

> **Analogy**: Imagine verifying a signed sentence where spaces are removed.  
>  
> Signed: `"pay bob 10"` → `"paybob10"`  
> Attacker submits: `"pay bo b10"` → `"paybob10"`  
>  
> Signature still matches, but meaning changed.

## Vulnerable Code Reference

### 1) Deployment (latest version)

```solidity
signedOnly(
    abi.encodePacked(msg.sender, templateName, initdata),
    signature
)
```

### 2) Deployment (specific version)

```solidity
signedOnly(
    abi.encodePacked(
        msg.sender,
        templateName,
        templateVersion,
        initdata
    ),
    signature
)
```

### 3) Instance call

```solidity
signedOnly(
    abi.encodePacked(msg.sender, instance, data),
    signature
)
```

### Root Cause

All cases pack **multiple user-controlled dynamic types** using `abi.encodePacked`.

## Impact

An attacker can craft alternative parameters that hash to the same value as a legitimately signed payload.

This may allow:

* Unauthorized template deployments  
* Unauthorized instance calls  
* Bypass of backend authorization  
* Potential fee bypass depending on flow  

Because signature validation is a primary security boundary, the impact is **Medium**.

## Recommended Mitigation

### ✅ Primary Fix — Use `abi.encode`

Replace all instances of:

```solidity
abi.encodePacked(...)
```

with:

```solidity
abi.encode(...)
```

`abi.encode` includes length prefixes, preventing boundary ambiguity.

**Safe example**

```solidity
signedOnly(
    abi.encode(msg.sender, templateName, initdata),
    signature
)
```

### ✅ Alternative — Sign structured hash off-chain

Compute the hash off-chain and pass a single `bytes32`:

```solidity
bytes32 digest = keccak256(
    abi.encode(msg.sender, templateName, initdata)
);
```

This avoids on-chain packing risks and reduces gas.

### ✅ Defense-in-Depth

* Avoid packing multiple dynamic types together.
* Prefer EIP-712 typed structured data for signatures.
* Add unit tests for collision scenarios.

## Pattern Recognition Notes

* **Packed Encoding with Multiple Dynamic Types**: Using `abi.encodePacked` on more than one dynamic input is a well-known collision hazard.
* **Signature Boundary Assumption**: Systems relying on off-chain signatures must ensure the signed message is uniquely encoded.
* **User-Controlled Encoding Surface**: When attackers control packed fields, collision feasibility increases significantly.
* **Prefer Structured Hashing**: EIP-712 or `abi.encode` should be the default for authentication flows.
* **Silent Auth Bypass Risk**: Unlike obvious reverts, signature collisions silently undermine trust assumptions.

## Quick Recall (TL;DR)

* **Bug**: `abi.encodePacked` used with multiple dynamic inputs in signature verification.  
* **Impact**: Hash collision allows reuse of valid signatures with different parameters → auth bypass.  
* **Fix**: Replace with `abi.encode` or structured hashing (EIP-712 preferred).