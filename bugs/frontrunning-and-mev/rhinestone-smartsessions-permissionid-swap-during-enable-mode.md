# PermissionID Swap Attack via Unsigned Permission Identifier in Enable-Mode Digest

* **Severity**: High
* **Source**: [Solodit](https://solodit.cyfrin.io/issues/enable-mode-can-be-frontrun-to-add-policies-for-a-different-permissionid-cantina-none-biconomy-pdf)
* **Affected Contract**: [SmartSession.sol](https://github.com/cantina-forks/rhinestone-smartsessions/blob/a6fc609c6357733e000a08705a99f613ce8841ed/contracts/SmartSession.sol)
* **Vulnerability Type**: Signature Forgery / Authorization Bypass / Privilege Escalation

## **Summary**

The SmartSession module supports an **ENABLE mode**, allowing a smart account to **create or update a session and use it within the same userOp**. To authorize this configuration, the account owner signs a special **EnableSession digest**, which includes session validator configuration and policy sets.

However:

### **The `permissionId` supplied in `userOp.signature` is *not* included in the signed digest, nor is it guaranteed to match the session data in `enableData`.**

Because ERC-4337 excludes the full signature field (`userOp.signature`) from `userOpHash`, a bundler or malicious user can **tamper with the permissionId** without changing the userOpHash.

As a result:

> A user who intends to enable policies for permission **X** can be front-run, and their policies will instead be installed onto **permission Y** — chosen by the attacker — allowing privilege escalation.

This attack allows a low-privilege session to steal policies intended for a high-privilege session.

## **A Better Explanation (With Simplified Example)**

### **Intended Behavior**

When enabling a session in SmartSession:

1. User signs an `EnableSession` payload (nested EIP-712) containing:

   * session validator configuration
   * policy sets (userOp, action, ERC1271)
   * session initialization data

2. SmartSession calculates:

   * `digest = hash(enableData, signerNonce, mode, account)`

3. SmartSession checks:

   * `account.isValidSignature(digest, enableSig)`
   * permissionId correctly computed from the session validator (in some branches)

4. SmartSession stores:

   * session validator
   * policy sets
   * marks `permissionId` as enabled

5. UserOp continues using the newly enabled session.

### **What Actually Happens (Bug)**

Two key issues:

#### **1) `permissionId` is *not* included in the signed digest**

The digest does not bind:

```text
permissionId ↔ enableData
```

#### **2) `permissionId` is not consistently verified against sessionToEnable data**

The check:

```solidity
if (permissionId != enableData.sessionToEnable.toPermissionIdMemory())
    revert InvalidPermissionId()
```

only runs when enabling a new session validator.
If the validator is already set → the check is skipped.

#### **Result**

A malicious bundler/front-runner can:

1. Observe a userOp intending to enable permission **X**
2. Replace the `permissionId` inside `userOp.signature` with **Y**
3. Because `userOp.signature` is not included in `userOpHash`,
   the verification still passes
4. SmartSession enables policies on **Y** instead of **X**

## **Why This Matters**

This allows **privilege escalation**:

A session with weak permissions (Y) can suddenly gain *all policies intended for a stronger session (X)*.

This breaks the core security assumption of SmartSession:

> "A session key can only receive the permissions intentionally authorized by the account owner."

Given that SmartSession is used in account-abstraction wallets, this can translate to:

* unauthorized contract calls becoming allowed
* unauthorized ERC20/ETH transfers becoming allowed
* session actions bypassing guardrails
* ERC1271 signature approvals leaking into the wrong session

## **Concrete Walkthrough (Alice & Mallory)**

### **Setup**

* Alice has two sessions:

  * `permissionId_X` → high-privilege session (can transfer tokens, interact with DeFi protocols)
  * `permissionId_Y` → low-privilege session (cannot move funds)

* Mallory controls a session key for `permissionId_Y`.

### **Alice wants to enable new policies for session X**

She signs an EnableSession userOp:

```text
enable policies for session X
signature = Sign(hash(enableData, nonce, mode, account))
```

The signature is **valid only for X**, in her mind.

### **Mallory front-runs the userOp**

Mallory modifies:

```text
permissionId: X → Y
```

inside `userOp.signature`.

This works because:

* `userOp.signature` is not hashed in `userOpHash`
* `permissionId` is not bound to the signed digest
* `_enablePolicies()` does not always enforce permissionId equivalence

### **SmartSession now enables these new powerful policies on Y**

Mallory's low-privilege session Y now becomes high-privilege.

Mallory can now:

* transfer Alice's tokens
* interact with DeFi contracts
* call arbitrary functions gated by action policies

All without Alice ever granting those permissions.

## **Vulnerable Code Reference**

### **1) permissionId extracted from userOp.signature (tamperable)**

```solidity
(SmartSessionMode mode, PermissionId permissionId, bytes calldata packedSig)
    = userOp.signature.unpackMode();
```

### **2) Signed digest does NOT include permissionId**

```solidity
bytes32 hash = enableData.getAndVerifyDigest(account, nonce, mode);
```

No binding between:

```text
permissionId ↔ enableData ↔ signature
```

### **3) permissionId consistency check is conditional**

```solidity
if (!_isISessionValidatorSet(permissionId, account)) {
    if (permissionId != enableData.sessionToEnable.toPermissionIdMemory()) {
        revert InvalidPermissionId(permissionId);
    }
}
```

If the validator is already set → this check is skipped → attacker bypass.

## **Recommended Mitigation**

### **1. Include `permissionId` in the enable digest**

In `getAndVerifyDigest`:

```solidity
hash = hash(account, nonce, mode, permissionId, enableData)
```

This binds signature → enableData → permissionId.

### **2. Always verify permissionId matches computed value**

Move:

```solidity
if (permissionId != enableData.sessionToEnable.toPermissionIdMemory())
    revert InvalidPermissionId(permissionId);
```

**outside** the conditional block so it runs on *every* enable.

### **3. Consider enforcing monotonic signerNonce per-permissionId**

This reduces exploitability even further.

### **4. Regression tests**

Add tests verifying:

* modifying permissionId causes immediate failure
* ENABLE mode always rejects mismatched permissionIds
* userOpHash tampering in userOp.signature cannot redirect permissions

## **Pattern Recognition Notes**

* **Missing Binding in Signed Digest**
  If critical identifiers (permissionId, roleId, configId) are not part of the signature digest, they can be swapped without breaking signatures.

* **ERC-4337 Pitfall: userOp.signature not included in userOpHash**
  Attackers can safely tamper with anything inside `userOp.signature` unless the contract re-hashes and verifies it internally.

* **Privileged Setup During Validation Phase**
  Any "enable-within-validate" flow must validate *all* parameters exhaustively, since it runs before execution.

* **Conditional Validation Loopholes**
  Security checks placed inside certain branches (e.g., only when creating a new validator) often create bypasses.

* **Privilege Escalation via Session Collisions**
  When multiple sessions exist, incorrect binding can allow a low-privilege session to inherit high privilege.

## **Quick Recall (TL;DR)**

* **Bug:** `permissionId` is not included in the signed digest, and equivalence to enableData isn't always checked.
* **Impact:** A malicious user/bundler can replace permissionId in `userOp.signature`, causing policies meant for X to be enabled for Y.
* **Effect:** Privilege escalation; attacker upgrades their own session with victim's policies.
* **Fix:** Bind permissionId to signed digest; always check permissionId consistency.
