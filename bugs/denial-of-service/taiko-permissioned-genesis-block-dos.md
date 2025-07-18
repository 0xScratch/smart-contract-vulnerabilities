# Denial of Service via Permissioned Genesis Block

- **Severity**: Medium  
- **Source**: [Code4rena](https://github.com/code-423n4/2024-03-taiko-findings/issues/274) / [One Bug Per Day](https://www.onebugperday.com/v1/1126)  
- **Affected Contract**: [LibProposing.sol](https://github.com/code-423n4/2024-03-taiko/blob/main/packages/protocol/contracts/L1/libs/LibProposing.sol)  
- **Vulnerability Type**: Denial of Service (DoS) - Access Control Logic Flaw

## Original Bug Description

> ### Vulnerability Details
>
> ### Description
>
> The `LibProposing.proposeBlock` function calls the `_isProposerPermitted` private function, to ensure if the `proposer is set`. Only that specific address has the permission to propose the block.
>
> In the `_isProposerPermitted` function, for the first block after the genesis block only the `proposerOne` is allowed to propose the first block as shown below:
>
> ```solidity
>   address proposerOne = _resolver.resolve("proposer_one", true);
>   if (proposerOne != address(0) && msg.sender != proposerOne) {
>       return false;
>   }
> ```
>
> But the issue here is that when the `msg.sender == proposerOne` the function `does not return true` if the following conditions occur.
>
> If the `proposer != address(0) && msg.sender != proposer`. In which case even though the `msg.sender == proposerOne` is `true` for the first block the `_isProposerPermitted` will still return `false` thus reverting the block proposer for the first block.
>
> Hence even though the `proposer_one` is the proposer of the first block the transaction will still revert if the above mentioned conditions occur and the `_isProposerPermitted` returns `false` for the first block after the genesis block.
>
> Hence this will break the block proposing logic since the proposal of the first block after the genesis block reverts thus not allowing subsequent blocks to be proposed.
>
> ### Proof of Concept
>
> ```solidity
>   TaikoData.SlotB memory b = _state.slotB;
>   if (!_isProposerPermitted(b, _resolver)) revert L1_UNAUTHORIZED();
> ```
>
> [https://github.com/code-423n4/2024-03-taiko/blob/main/packages/protocol/contracts/L1/libs/LibProposing.sol#L93-L94](https://github.com/code-423n4/2024-03-taiko/blob/main/packages/protocol/contracts/L1/libs/LibProposing.sol#L93-L94)
>
> ```solidity
>   function _isProposerPermitted(
>       TaikoData.SlotB memory _slotB,
>       IAddressResolver _resolver
>   )
>       private
>       view
>       returns (bool)
>   {
>       if (_slotB.numBlocks == 1) {
>           // Only proposer_one can propose the first block after genesis
>           address proposerOne = _resolver.resolve("proposer_one", true);
>           if (proposerOne != address(0) && msg.sender != proposerOne) {
>               return false;
>           }
>          
>       address proposer = _resolver.resolve("proposer", true);
>       return proposer == address(0) || msg.sender == proposer;
>   }
> ```
>
> [https://github.com/code-423n4/2024-03-taiko/blob/main/packages/protocol/contracts/L1/libs/LibProposing.sol#L299-L317](https://github.com/code-423n4/2024-03-taiko/blob/main/packages/protocol/contracts/L1/libs/LibProposing.sol#L299-L317)
>
> ### Tools Used
>
> Manual Review and VSCode
>
> ### Recommended Mitigation Steps
>
> It is recommended to add logic in the `LibProposing._isProposerPermitted` function to `return true` when the `msg.sender == proposerOne` for proposing the first block after the genesis block.

## Summary

A logic flaw in the access control for proposing the first Layer 2 block on Taiko can prevent even the designated `proposer_one` address from successfully proposing the genesis block if a different `proposer` is set for subsequent blocks. This creates a Denial of Service (DoS) condition: if the contract configuration uses both `proposer_one` and a separate `proposer`, the first L2 block cannot be proposed by anyone, halting all further chain progress.

## A Better Explanation (With Example)

Before proceeding to the example, we must know few things going on in here, especially within the `proposeBlock` function:

- The `proposeBlock` function is responsible for proposing a new block to the Taiko chain. Internally this function also calls `_isProposerPermitted` which checks whether the sender is allowed to propose a block or not.
- Now, here are two main scenarios when proposing a block:
  - Sometimes, a specific address called `proposer` is having the responsibility of proposing blocks. Thus, no one else can propose blocks during that scenario.
  - At other times, if no one is stated as the `proposer` then anyone can propose the blocks. Simple and straight!
- But, there's an exception when it comes to "proposing the first block after the genesis block", none of the above two scenarios apply in this condition, and hence, a special address called `proposer_one` is having the responsibility to propose the first block.

If you are able to understand the above context, you are on right path...

Moving to the example of **creating the first block**, it will have two scenarios to make you realize where the code actually breaks.

### ✅ Scenario 1: proposer is not set (i.e. address(0))

Let's consider some variables here:

- `proposer_one` = Alice
- `proposer` = `empty` i.e. `address(0)`
- `msg.sender` = Alice

Now step-by-step:

As we are creating the first block, hence, we will jump into the condition: `if (_slotB.numBlocks == 1)`

```solidity
if (proposerOne != address(0) && msg.sender != proposerOne)
```

- `proposerOne != address(0)` → TRUE ✅
- `msg.sender != proposerOne` → FALSE ❌
- ==> TRUE && FALSE = FALSE → condition does not trigger, so the function keeps running ✔️

Now the last return statement:

```solidity
return proposer == address(0) || msg.sender == proposer;
```

- `proposer == address(0)` → TRUE ✅
- `msg.sender == proposer` -> TRUE ✅ (We won't even need to check this as the first condition is already TRUE)
- ==> Overall expression returns TRUE ✔️

**✅ So Alice (proposer_one) CAN propose the block in this case.**

### ❌ Scenario 2: proposer is set to someone else (e.g. Bob)

Now, let's again consider some variables:

- `proposer_one` = Alice
- `proposer` = Bob
- `msg.sender` = Alice

The same first check:

```solidity
if (proposerOne != address(0) && msg.sender != proposerOne)
```

- `proposerOne != address(0)` → TRUE ✅
- `msg.sender != proposerOne` → FALSE ❌
- ==> TRUE && FALSE = FALSE → condition does not trigger, so the function keeps running ✔️

Now the last return statement:

```solidity
return proposer == address(0) || msg.sender == proposer;
```

- `proposer == address(0)` → FALSE ❌
- `msg.sender == proposer` -> FALSE ❌
- ==> Overall expression returns FALSE ❌

**⛔ Alice is denied the ability to propose the first block, even though she's `proposer_one`. That's the bug!**

## Real-World Context

- **Genesis Block Criticality**: Without the successful proposal of the first block, the entire Taiko chain cannot initialize or process any Layer 2 transactions.
- **Setup Flexibility**: Protocol teams often want to restrict the first block to a special address (for coordinated launch or audit reasons), then hand off control to a more decentralized or production address. This bug undermines that pattern.
- **Chain Liveness**: Progression of the layer is blocked, preventing user onboarding, dApp deployment, and normal economic activity.

## Key Details

- **Pattern**: First-block permission intersecting general proposer rule without proper bypass.
- **Root Cause**: Even when the special `proposer_one` address rightfully tries to propose the first block, the code falls through to generic proposer checks and blocks them if a different `proposer` is set.
- **Risk**: Chain initialization can be completely blocked, resulting in a critical liveness failure.

## Vulnerable Code Reference

```solidity
function _isProposerPermitted(
    TaikoData.SlotB memory _slotB,
    IAddressResolver _resolver
)
    private
    view
    returns (bool)
{
    if (_slotB.numBlocks == 1) {
        // Only proposer_one can propose the first block after genesis
        address proposerOne = _resolver.resolve("proposer_one", true);
        if (proposerOne != address(0) && msg.sender != proposerOne) {
            return false;
        }
    }
    address proposer = _resolver.resolve("proposer", true);
    return proposer == address(0) || msg.sender == proposer;
}
```

## Impact

- **DoS**: The chain cannot progress beyond genesis if the genesis block cannot be proposed.
- **Operational Risk**: Misconfiguration (using different addresses for `proposer_one` and `proposer`) can permanently halt deployment, even with correct off-chain intentions.

## Mitigation

- **Bypass Check for proposer_one**: After confirming the sender is `proposer_one` for the first block, immediately return `true`, skipping the generic check.
- **Example Patch**:

```solidity
if (_slotB.numBlocks == 1) {
    address proposerOne = _resolver.resolve("proposer_one", true);
    if (proposerOne != address(0)) {
        if (msg.sender == proposerOne) return true;
        return false;
    }
}
```

- **Testing**: Always test genesis permission logic with all intended combinations of `proposer_one` and `proposer` values.

## Pattern Recognition Notes

- Look for **multiple permission roles** (`proposer_one`, `proposer`) and missing short-circuit logic that may cause valid roles to be rejected.
- Ensure **special-case accounts have clear positive pathways** (`return true`) separate from general logic.
- Watch for hidden interdependencies between configuration values that restrict initialization flow.
- **Red Flags**:
  - "if ... && ..." without a clear else or early return in permission logic
  - Overlapping or nested permission rules
  - "Initializer"/"bootstrapper" roles dangling after setup
  - No code path that guarantees the first essential action succeeds under non-default configurations
- **Common locations**: block proposal logic, upgrade initialization, bootstrap/migration handlers.
- **Testing approach**: simulate real deployments with different role combinations and assert access for each.
