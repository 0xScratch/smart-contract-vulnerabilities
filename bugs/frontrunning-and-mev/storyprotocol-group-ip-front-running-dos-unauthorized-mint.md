# Front-Running DoS on Group IP Setup via Unauthorized License Token Minting

* **Severity**: High
* **Source**: [Solodit](https://solodit.cyfrin.io/issues/griefing-attack-in-group-ip-management-via-license-token-minting-halborn-story-proof-of-creativity-protocol-markdown)
* **Affected Contracts**:
  * [GroupingModule.sol](https://github.com/storyprotocol/protocol-core-v1/blob/main/contracts/modules/grouping/GroupingModule.sol#L250-L257) (registerGroup, addIp)
  * [LicensingModule.sol](https://github.com/storyprotocol/protocol-core-v1/blob/main/contracts/modules/licensing/LicensingModule.sol#L160-L236) (mintLicenseTokens - missing permission check)
* **Vulnerability Type**: Denial of Service (DoS) / Front-running / Missing Access Control / Griefing

## Summary

The `registerGroup` function in `GroupingModule` allows anyone to create a new Group IP Asset (a bundle that can collect multiple IP Assets). After creation (and before or during license attachment), the legitimate owner intends to attach license terms and then add individual IPs to the group using `addIp`.

However, `mintLicenseTokens` in `LicensingModule` originally lacked any permission check (no `verifyPermission` modifier or owner-only restriction). Any user could front-run the owner and mint even **one license token** for the freshly created Group IP.

This sets `getTotalTokensByLicensor(groupId) > 0`, which causes the internal lock check `_checkIfGroupMembersLocked` to revert in `addIp` (and similarly in `removeIp`). Because the check happens before any state update, the owner can **never** add/remove IPs → the Group IP becomes permanently unusable (DoS on group composition).

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Register Group**: Owner calls `registerGroup` → mints Group NFT → registers empty Group IP Asset.
2. **Attach License Terms**: Owner calls `attachLicenseTerms(groupId, template, termsId)`.
3. **Add IPs to Group**: Owner calls `addIp(groupId, ipIds[])`  
   → But only allowed if:
     * No derivative IPs exist, **and**
     * `LICENSE_TOKEN.getTotalTokensByLicensor(groupId) == 0` (no tokens minted yet)

   This check prevents adding members after the group has been "activated" for licensing/derivatives.

### What Actually Happens (Bug)

* `mintLicenseTokens(licensorIpId, ...)` had **no access control** — anyone could call it for **any** IP, including a brand-new Group IP.
* Attacker monitors mempool → sees group registration tx → front-runs owner's next steps (attach terms or add IPs) → mints 1 token for the group.
* Now `total minted > 0` → `_checkIfGroupMembersLocked` reverts on every `addIp` / `removeIp` call.
* Revert happens **before** any state change → owner stuck forever; group is bricked.

### Why This Matters

* A single malicious (or even accidental) mint bricks the entire Group IP for composition.
* Groups are core for bundling/remixing IPs → breaks key composability feature.
* Griefing attack: cheap (just gas + small mint fee), high impact, no direct theft but destroys value for creator.
* Especially painful in high-throughput environments where front-running is easy.

### Concrete Walkthrough (Alice & Attacker)

* **Setup**: Alice registers fresh Group IP (`groupId`).
* **Alice attaches terms** (or plans to add IPs next).
* **Attacker front-runs**: Calls `mintLicenseTokens(groupId, template, termsId, 1, attacker, "", 0)`.
* **Alice tries `addIp`** → `_checkIfGroupMembersLocked` sees `getTotalTokensByLicensor(groupId) = 1` → reverts with `GroupFrozenDueToAlreadyMintLicenseTokens`.
* Every future attempt hits the same check → **permanent DoS** on group management.

> **Analogy**: Imagine creating an empty shared folder (Group IP), planning to add files later. But before you add anything, someone else "checks out" a single blank permission slip for your folder. Now the folder is marked "in use" forever — you can never add your own files, even though you own it.

## Vulnerable Code Reference (Pre-Fix)

**1) No permission check in `mintLicenseTokens`**:

    ```solidity
    // LicensingModule.sol (vulnerable version)
    function mintLicenseTokens(
        address licensorIpId,
        address licenseTemplate,
        uint256 licenseTermsId,
        uint256 amount,
        address receiver,
        bytes calldata hookData,
        uint256 maxMintingFee
    ) external { 
        // ... no verifyPermission(licensorIpId) or owner check here
        // Proceeds to mint even if caller is not owner
    }
    ```

**2) Lock check that bricks group on minted tokens**:

    ```solidity
    // GroupingModule.sol
    function _checkIfGroupMembersLocked(address groupIpId) internal view {
        if (LICENSE_REGISTRY.hasDerivativeIps(groupIpId)) {
            revert Errors.GroupingModule__GroupFrozenDueToHasDerivativeIps(groupIpId);
        }
        if (LICENSE_TOKEN.getTotalTokensByLicensor(groupIpId) > 0) {
            revert Errors.GroupingModule__GroupFrozenDueToAlreadyMintLicenseTokens(groupIpId);
        }
    }
    ```

**3) `addIp` relies on the above check**:

    ```solidity
    function addIp(address groupIpId, address[] memory ipIds) external {
        // ...
        _checkIfGroupMembersLocked(groupIpId);  // reverts if poisoned
        // Never reaches actual add logic
    }
    ```

## Recommended Mitigation (Actual Fix Implemented)

The Story team fixed this by adding owner control via a **`disabled`** flag in `LicensingConfig`:

**1) Add `disabled` field to `LicensingConfig` struct**:

    ```solidity
    struct LicensingConfig {
        bool isSet;
        uint256 mintingFee;
        address licensingHook;
        bytes hookData;
        uint32 commercialRevShare;
        bool disabled;  // <--- new flag
    }
    ```

**2) Check `disabled` before minting / registering derivatives**:

    ```solidity
    // In mintLicenseTokens and registerDerivative
    if (lsc.isSet && lsc.disabled) {
        revert Errors.LicensingModule__LicenseDisabled(licensorIpId, licenseTemplate, licenseTermsId);
    }
    ```

**3) Owner can now**:

* After creating group + attaching terms → call `setLicensingConfig` to set `disabled = true` (blocks all mints).
* Safely add/remove IPs without interference.
* Later set `disabled = false` when ready for public licensing.

This gives the owner a "setup mode" window → perfect defense against front-running griefers.

(Implemented in commit: <https://github.com/storyprotocol/protocol-core-v1/commit/ceda981b48e9cbf4ce45aa7fa747cb6360396a5c>)

## Pattern Recognition Notes

* **Missing Access Control on Critical Actions**: Minting tokens affects downstream logic (locking) → must restrict to owner/operator.
* **Front-Running Griefing Window**: Between creation and setup completion, unauthenticated actions can poison state irreversibly.
* **State-Based Locks Without Escape Hatches**: Checks like `totalTokens > 0` are irreversible once poisoned → design for owner overrides or skip logic.
* **Defensive Defaults**: Start with features disabled/locked; owner explicitly enables when ready.
* **Mempool Race Conditions**: Common in public blockchains → use commit-reveal, timestamps, or owner-gated phases for sensitive setup.

## Quick Recall (TL;DR)

* **Bug**: Anyone could mint license tokens for a new Group IP → sets minted count > 0 → permanently blocks `addIp`/`removeIp` via lock check.
* **Impact**: Group IP becomes unusable for composition → **DoS on core grouping feature**.
* **Fix**: Add `disabled` flag in `LicensingConfig` → owner can block minting during setup, toggle when ready.
* **Lesson**: Always protect state-changing actions that trigger irreversible locks with strict access control.
