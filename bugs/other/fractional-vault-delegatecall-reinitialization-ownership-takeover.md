# Delegatecall Reinitialization via Nonce Reset in Vault

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-07-fractional-findings/issues/487)
* **Affected Contract**: [Vault.sol](https://github.com/code-423n4/2022-07-fractional/blob/main/src/Vault.sol)
* **Vulnerability Type**: Delegatecall Storage Corruption / Ownership Takeover / Re-Initialization Attack

## Summary

The `Vault.execute()` function performs a **delegatecall** to external target contracts after verifying permissions using a Merkle proof or ownership check.

Although the contract attempts to protect ownership by verifying that the `owner` variable remains unchanged after the delegatecall, it **fails to protect other critical storage variables**, particularly the `nonce`.

Because `nonce` controls initialization, a malicious plugin contract can reset `nonce` to `0` during delegatecall, allowing the `init()` function to be called again. This results in **Vault ownership takeover**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The Vault initialization flow is designed to occur only once:

```solidity
function init() external {
    if (nonce != 0) revert;
    nonce = 1;
    owner = msg.sender;
}
```

Once initialized:

* `nonce = 1`
* `init()` becomes permanently locked
* Ownership remains secure

Additionally, when executing plugin calls:

```solidity
(success, response) = _target.delegatecall(_data);
```

The contract attempts to prevent ownership takeover:

```solidity
address owner_ = owner;

delegatecall(...)

if (owner_ != owner) revert;
```

This ensures plugins cannot change the `owner`.

### What Actually Happens (Bug)

The contract only protects `owner`, but **does not protect `nonce`**.

A malicious plugin can:

1. Execute via delegatecall
2. Modify Vault storage
3. Reset `nonce = 0`

Once `nonce = 0`, the `init()` function becomes callable again.

This allows the attacker to **reinitialize the Vault and become owner**.

## Why This Matters

* Delegatecall allows full storage access
* Only owner variable protected
* Initialization depends on nonce
* Ownership takeover becomes possible

This results in:

* Ownership takeover
* Full Vault control
* Asset theft potential

## Concrete Walkthrough (Alice & Mallory)

### Setup

Vault initialized:

```text
owner = Alice
nonce = 1
```

### Mallory Attack

Mallory deploys malicious plugin:

```solidity
contract HackyTargetContract {
    address public gap_owner;
    bytes32 public gap_merkleRoot;
    uint256 public nonce;

    function changeNonce() public {
        nonce = 0;
    }
}
```

### Step-by-Step Attack

**Step 1 — Plugin Permissioned**

Owner adds malicious contract to Merkle tree.

**Step 2 — Execute Plugin**

```solidity
vault.execute(maliciousContract)
```

Delegatecall executes:

```text
nonce = 0
```

Vault state becomes:

```text
owner = Alice
nonce = 0
```

**Step 3 — Reinitialize Vault**

Attacker calls:

```solidity
vault.init()
```

Now:

```text
owner = attacker
nonce = 1
```

Ownership stolen.

## Analogy

Think of Vault initialization as:

> A door that can only be locked once.

The contract locks the door using `nonce`.

The attacker resets the lock (`nonce = 0`) using delegatecall.

Now the attacker locks the door again — but **with themselves inside**.

## Vulnerable Code Reference

### 1) Delegatecall execution

```solidity
(success, response) = _target.delegatecall{gas: stipend}(_data);
```

Vault allows target contract to modify storage.

### 2) Owner-only protection

```solidity
address owner_ = owner;

(success, response) = _target.delegatecall(_data);

if (owner_ != owner) revert OwnerChanged(owner_, owner);
```

Only `owner` is checked.

### 3) Re-initialization logic

```solidity
function init() external {
    if (nonce != 0) revert;
    nonce = 1;
    owner = msg.sender;
}
```

Resetting nonce enables re-initialization.

## Storage Layout Exploit

Vault storage:

```text
slot 0 → owner
slot 1 → merkleRoot
slot 2 → nonce
```

Attacker contract mirrors layout:

```text
slot 0 → gap_owner
slot 1 → gap_merkleRoot
slot 2 → nonce
```

Because of delegatecall:

```text
nonce = 0
```

Actually modifies Vault storage.

## Recommended Mitigation

### 1) Protect nonce variable

```solidity
address owner_ = owner;
uint256 nonce_ = nonce;

(success, response) = _target.delegatecall(_data);

if (owner_ != owner || nonce_ != nonce) {
    revert InvalidStateChange();
}
```

### 2) Avoid delegatecall to untrusted contracts

Prefer:

* call instead of delegatecall
* strict plugin review

### 3) Use immutable initialization flag

Example:

```solidity
bool initialized;
```

More explicit than nonce.

### 4) Add invariant checks

Ensure:

* owner unchanged
* nonce unchanged
* merkleRoot unchanged

## Pattern Recognition Notes

### Delegatecall Storage Corruption

Delegatecall gives target full storage access.
Always protect critical variables.

### Incomplete State Validation

Only checking one variable (`owner`) leaves other attack surfaces.

### Initialization Re-Entry

Initialization functions should never be reachable again.

### Storage Layout Manipulation

Attackers align storage layout to overwrite specific slots.

### Delegatecall Risk Pattern

Common in:

* Proxy contracts
* Plugin systems
* Smart wallets

## Quick Recall (TL;DR)

* **Bug**: Delegatecall allows resetting `nonce`
* **Impact**: Vault can be re-initialized
* **Result**: Ownership takeover
* **Fix**: Validate `nonce` after delegatecall
