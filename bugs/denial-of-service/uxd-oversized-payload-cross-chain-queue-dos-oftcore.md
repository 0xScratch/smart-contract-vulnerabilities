# Cross-Chain Channel DoS via Oversized Payload in OFTCore

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-01-uxd-judging/issues/270)
* **Affected Contract**: [OFTCore.sol](https://github.com/sherlock-audit/2023-01-uxd/blob/main/contracts/external/layer-zero/token/oft/OFTCore.sol#L31-L33)
* **Vulnerability Type**: Denial of Service (DoS) / Input Validation / Cross-Chain Queue Poisoning

## Summary

`OFTCore#sendFrom` allows users to supply an arbitrary-length `_toAddress` (bytes). This value is directly embedded into the LayerZero payload without any size restriction.

An attacker can exploit this by crafting an **excessively large `_toAddress`**, resulting in a payload that **cannot be executed on the destination chain due to gas limits**. While the message is successfully sent from the source chain, it becomes **impossible to process on the destination chain**.

Because LayerZero enforces **strict nonce ordering**, a single unexecutable message causes all subsequent messages to fail, effectively **permanently blocking the cross-chain communication channel** and leading to **loss of user funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User calls `sendFrom` on source chain**

   * Tokens are debited (burned/locked)
   * Payload is created and sent via LayerZero

2. **Destination chain receives message**

   * `_nonblockingLzReceive` processes payload
   * `_creditTo` mints/unlocks tokens to recipient

3. **Messages are processed sequentially (by nonce)**

### What Actually Happens (Bug)

* `_toAddress` is user-controlled and has **no size limit**
* It is encoded into payload:

```solidity
bytes memory lzPayload = abi.encode(PT_SEND, _toAddress, amount);
```

* Attacker submits extremely large `_toAddress`
* Payload becomes **too large to execute** on destination chain

### Why This Breaks the System

* Source chain (e.g., Arbitrum) → high gas limit → transaction succeeds ✅
* Destination chain (e.g., Optimism) → lower gas limit → execution fails ❌

Now the critical part:

```solidity
require(_nonce == ++inboundNonce[_srcChainId][_srcAddress])
```

👉 Messages **must be processed in order**

### Concrete Walkthrough (Alice & Mallory)

* **Setup**:

  * Channel: Arbitrum → Optimism
  * `inboundNonce = 0`

#### **Mallory (attacker)**

1. Calls `sendFrom` with:

   ```text
   _toAddress = extremely large bytes
   ```
2. Payload becomes huge
3. Message sent with:

   ```text
   nonce = 1
   ```
4. On destination:

   * Execution fails due to gas exhaustion
   * Message **never succeeds**

#### **Alice (victim)**

1. Calls `sendFrom` normally
2. Her message gets:

   ```text
   nonce = 2
   ```

---

#### **Now on destination chain**

When processing Alice's message:

```text
Expected nonce = 1
Incoming nonce = 2
```

👉 Fails nonce check → revert ❌

### Final Result

* Message #1 → stuck forever
* Message #2+ → cannot be processed
* Entire channel → **permanently blocked**

### Why This Matters

* A single malicious transaction can **brick the entire bridge route**
* All users sending tokens afterward:

  * Tokens debited on source chain ✅
  * Never credited on destination ❌
* Results in **permanent fund loss and protocol shutdown for that route**

### Key Insight

> Blocking occurs **even if payload is never stored**

The DoS arises from **nonce ordering**, not just LayerZero's stored payload mechanism.

### Analogy

A single-lane bridge where cars must pass in order:

* Attacker places an unmovable truck in the middle 🚛
* No car behind can pass
* Traffic is permanently blocked

## Vulnerable Code Reference

**1) Unbounded user input in `sendFrom`**

```solidity
function sendFrom(..., bytes calldata _toAddress, ...) public payable {
    _send(..., _toAddress, ...);
}
```

**2) Payload construction using unbounded input**

```solidity
bytes memory lzPayload = abi.encode(PT_SEND, _toAddress, amount);
```

**3) Strict nonce ordering in LayerZero endpoint**

```solidity
require(_nonce == ++inboundNonce[_srcChainId][_srcAddress], "LayerZero: wrong nonce");
```

**4) Cross-chain send logic**

```solidity
_lzSend(_dstChainId, lzPayload, ...);
```

## Recommended Mitigation

1. **Enforce size limits on `_toAddress` (primary fix)**

```solidity
require(_toAddress.length <= 32, "Invalid address length");
```

* EVM address: 20 bytes
* Solana/Aptos: 32 bytes

2. **General payload size validation**

Ensure total payload size remains within safe execution bounds for all supported chains.

3. **Cross-chain safety checks**

* Validate assumptions about destination gas limits
* Avoid user-controlled data directly influencing payload size

4. **Fail-safe mechanisms**

* Consider allowing message skipping or retry mechanisms
* Add admin recovery for stuck nonces (if architecture permits)

## Pattern Recognition Notes

* **Unbounded Input in Cross-Chain Payloads**
  User-controlled data included in cross-chain messages must always be size-restricted.

* **Nonce-Based Queue Poisoning**
  Systems enforcing strict sequential execution are vulnerable to **single-message permanent DoS**.

* **Cross-Chain Gas Asymmetry**
  Different chains have different gas limits — assumptions of executability can break.

* **Implicit Blocking via Ordering**
  Even without explicit storage of failed messages, strict ordering alone can halt systems.

* **Ingress Validation is Critical**
  Always validate inputs **before** cross-chain transmission — failures on destination are far more dangerous.

## Quick Recall (TL;DR)

* **Bug**: `_toAddress` has no size limit → attacker creates huge payload
* **Impact**: Destination execution fails → nonce stuck → entire channel blocked
* **Root Cause**: Strict nonce ordering + unbounded user input
* **Fix**: Restrict `_toAddress` length (≤ 32 bytes)
