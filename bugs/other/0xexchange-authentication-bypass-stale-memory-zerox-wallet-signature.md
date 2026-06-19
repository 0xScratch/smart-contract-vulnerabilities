# Authentication Bypass via Stale Memory in Wallet Signature Verification

* **Severity**: Critical
* **Source**: [The 0x Vulnerability, Explained (samczsun)](https://samczsun.com/the-0x-vulnerability-explained/)
* **Affected Contract**: [MixinSignatureValidator.sol](https://github.com/0xProject/0x-monorepo/blob/965d6098294beb22292090c461151274ee6f9a26/packages/contracts/src/2.0.0/protocol/Exchange/MixinSignatureValidator.sol?ref=samczsun.com#L233-L273)
* **Vulnerability Type**: Authentication Bypass / Signature Forgery / Uninitialized Memory Usage / Unsafe Low-Level Call Handling

## Summary

0x Exchange v2 supported a special **Wallet signature type** that validated signatures by calling:

```solidity
wallet.isValidSignature(hash, signature)
```

using a low-level `staticcall`.

The implementation assumed that a successful call would return a 32-byte boolean value indicating whether the signature was valid.

However, two subtle EVM behaviors combined to create a critical authentication bypass:

1. **Calls to EOAs (accounts with no code) succeed instead of failing.**
2. **CALL/STATICCALL do not clear the output memory region before copying return data.**

As a result, an attacker could provide a signer address that was an ordinary EOA and submit a signature consisting only of the Wallet signature type byte (`0x04`).

The call would succeed, return **zero bytes**, leave memory unchanged, and the contract would interpret stale calldata already present in memory as a non-zero boolean value (`true`).

This caused the Exchange to accept **arbitrary orders without a real signature**, allowing attackers to execute trades on behalf of any EOA that had approved the Exchange contract.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The Exchange supports multiple signature formats.

For Wallet signatures, verification works as follows:

1. User submits an order.
2. Exchange extracts the signature type.
3. If the type is `Wallet`, Exchange calls:

```solidity
wallet.isValidSignature(hash, signature)
```

4. Wallet returns:

```solidity
true  -> signature valid
false -> signature invalid
```

5. Exchange accepts the order only if `true` is returned.

Conceptually:

```solidity
bool valid = wallet.isValidSignature(hash, sig);
require(valid);
```

### What Actually Happens (Bug)

The implementation used:

```solidity
staticcall(...)
```

and instructed the EVM to write return data directly over the calldata buffer.

The developers assumed:

```text
successful call
    ->
32-byte return value
    ->
memory overwritten
    ->
safe to read
```

However:

### Step 1 — Calling an EOA does NOT fail

Suppose the signer address is:

```text
Alice (EOA)
```

instead of a smart contract wallet.

Most developers expect:

```text
call EOA
    ->
failure
```

but the EVM actually does:

```text
call EOA
    ->
success
    ->
return 0 bytes
```

because executing outside contract code is treated as an implicit `STOP`.

### Step 2 — No Return Data Means No Memory Overwrite

The contract expects:

```text
32 bytes
```

of return data.

The EOA returns:

```text
0 bytes
```

The EVM therefore copies:

```text
min(32, 0) = 0 bytes
```

into memory.

Nothing is overwritten.

The output buffer still contains the original calldata.

### Step 3 — Stale Memory Is Interpreted as a Boolean

The contract then executes:

```solidity
isValid := mload(outputLocation)
```

and loads the first 32 bytes from memory.

Those bytes are not return data.

They are leftover calldata such as:

```text
0x1626ba7eaaaaaaaaaaaaaaaa...
```

which is non-zero.

When converted to a boolean:

```solidity
bool(nonZeroValue)
```

becomes:

```solidity
true
```

The function therefore returns:

```solidity
true
```

even though no signature verification occurred.

### Why This Matters

The signature verification mechanism was the only thing preventing attackers from submitting fake orders.

Because the Wallet verification path always returned `true` for EOAs:

* Any EOA could be impersonated.
* No cryptographic signature was required.
* Any account that had approved the Exchange could have its orders forged.
* Attackers could execute trades on behalf of arbitrary users.

This effectively broke the core security assumption of the protocol.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

Alice is an ordinary EOA.

She previously approved the Exchange contract to spend her tokens.

```text
Alice
 ├─ Tokens
 └─ Exchange Approval
```

#### Mallory's Attack

Mallory creates a fake order:

```text
Sell Alice's tokens
Receive worthless assets
```

Mallory provides:

```text
signature = 0x04
```

which corresponds to:

```text
SignatureType.Wallet
```

and contains no actual signature data.

#### Exchange Verification

The Exchange enters the Wallet verification path:

```solidity
isValidWalletSignature(...)
```

and performs:

```solidity
staticcall(Alice)
```

where Alice is an EOA.

#### EVM Behavior

The call:

```text
succeeds
```

but returns:

```text
0 bytes
```

Memory remains unchanged.

The contract reads stale calldata from memory:

```text
0x1626ba7eaaaa....
```

which is non-zero.

#### Result

The loaded value becomes:

```solidity
true
```

and Exchange concludes:

```text
Alice signed this order
```

even though she never signed anything.

The forged trade can now execute.

> **Analogy:** Imagine a security guard expects a visitor badge to be returned after checking a person. Instead, nobody returns a badge at all. Rather than noticing the absence, the guard accidentally reuses an old badge already sitting on the desk and treats it as proof of authorization.

## Vulnerable Code Reference

### 1) Low-Level Call to Wallet Address

```solidity
let success := staticcall(
    gas,
    walletAddress,
    cdStart,
    mload(calldata),
    cdStart,
    32
)
```

The contract assumes that a successful call implies valid return data exists.

### 2) No Check That Target Contains Code

```solidity
staticcall(walletAddress, ...)
```

No verification is performed that:

```solidity
extcodesize(walletAddress) > 0
```

Therefore EOAs are accepted as wallet contracts.

### 3) No Check on Return Data Size

```solidity
isValid := mload(cdStart)
```

The contract immediately loads memory without verifying:

```solidity
returndatasize() == 32
```

As a result, stale memory may be interpreted as return data.

### 4) Reuse of Input Buffer as Output Buffer

```solidity
cdStart
```

is used as both:

```text
input location
output location
```

meaning old calldata remains present if no bytes are returned.

## Root Cause Analysis

This vulnerability exists because two independent assumptions were incorrect:

### Assumption #1

```text
Calling an EOA will fail.
```

Reality:

```text
Calling an EOA succeeds and returns zero bytes.
```

### Assumption #2

```text
Output memory will contain zeroes if no data is returned.
```

Reality:

```text
CALL/STATICCALL never clear memory.
Only returned bytes are copied.
```

Combining both behaviors produces:

```text
EOA Call
    ->
Success
    ->
0 Return Bytes
    ->
Memory Unchanged
    ->
Stale Calldata Read
    ->
Non-Zero Value
    ->
true
```

which bypasses authentication.

## Recommended Mitigation

### 1. Verify Target Contains Code

Reject EOAs before performing wallet signature validation.

```solidity
if (extcodesize(walletAddress) == 0) {
    revert("WALLET_ERROR");
}
```

This matches compiler-generated safety checks.

### 2. Verify Exact Return Data Size

Require the expected ABI-encoded boolean response.

```solidity
if (returndatasize() != 32) {
    revert("WALLET_ERROR");
}
```

This prevents stale memory from being interpreted as valid output.

### 3. Avoid Reading Memory Without Validation

Never perform:

```solidity
mload(outputPtr)
```

unless return size has already been validated.

### 4. Prefer Solidity Wrappers When Possible

Compiler-generated external calls automatically include:

* Code existence checks.
* Return-size validation.
* ABI decoding protections.

These safeguards eliminate entire classes of low-level call bugs.

### 5. Initialize or Clear Output Buffers

If low-level assembly is required:

```solidity
mstore(outputPtr, 0)
```

before the call.

While not a complete fix, it reduces stale-memory risk.

## Pattern Recognition Notes

### Low-Level Call Return Data Validation

Whenever you see:

```solidity
call(...)
staticcall(...)
delegatecall(...)
```

followed by:

```solidity
mload(...)
```

immediately ask:

> What happens if fewer bytes are returned than expected?

### Stale Memory Usage

A common EVM bug pattern:

```text
Read memory
without proving
that memory was written
during the current operation.
```

### EOA vs Contract Confusion

Never assume:

```solidity
call(address)
```

fails when the address is not a contract.

Always verify:

```solidity
extcodesize(address) > 0
```

when contract behavior is expected.

### Authentication Through External Calls

Authentication systems relying on external contract responses must verify:

* Target existence.
* Return size.
* Return format.
* Return semantics.

Failure in any of these can become a signature bypass.

### Gas Optimizations Can Introduce Security Bugs

Reusing calldata memory as output memory saved gas but removed an implicit safety margin.

Optimization should never bypass validation assumptions.

## Auditor Checklist

When reviewing low-level calls, check:

* [ ] Is the target guaranteed to be a contract?
* [ ] Is `extcodesize()` validated?
* [ ] Is `returndatasize()` checked?
* [ ] Can the callee return zero bytes?
* [ ] Can the callee return fewer bytes than expected?
* [ ] Is stale memory being read?
* [ ] Is ABI decoding performed safely?
* [ ] Is the result used for authentication or authorization?

If any answer is uncertain, investigate further.

### Quick Recall (TL;DR)

* **Bug:** Wallet signature verification trusted stale memory after a successful call that returned zero bytes.
* **Root Cause #1:** Calls to EOAs succeed rather than fail.
* **Root Cause #2:** CALL/STATICCALL do not clear output memory before copying return data.
* **Impact:** Any EOA could be impersonated using a signature consisting only of `0x04` (Wallet signature type).
* **Fix:** Require target code existence, require `returndatasize() == 32`, and never read output memory without validating returned data first.
* **Lesson:** Always validate both the callee and the returned data when using low-level EVM calls.
