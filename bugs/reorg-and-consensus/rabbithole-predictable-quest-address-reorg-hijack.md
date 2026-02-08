# Predictable Quest Address via CREATE Opcode - Reorg Attack Vulnerability

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/661)
* **Affected Contract**: [QuestFactory.sol](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol)
* **Vulnerability Type**: Predictable Address / Front-Running / Reorganization (Reorg) Attack / Deployment Risk

## Summary

The `createQuest` function in `QuestFactory` deploys new quest contracts (ERC20Quest or ERC1155Quest) using the standard `new` keyword, which relies on the **CREATE** opcode.  
This makes the deployed address **deterministic and predictable** — it depends solely on the QuestFactory's address + its current nonce (transaction/deployment counter).  

On chains prone to **block reorganizations** (reorgs) like Polygon, Optimism, and Arbitrum, an attacker can exploit a reorg window to **hijack** the predictable address:  
They deploy their own quest to the same address during the reorg, causing the original creator's subsequent funding transaction to send reward tokens to the attacker's quest contract instead.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Quest Creation**: A project calls `createQuest(...)` → QuestFactory deploys a new quest contract using `new Erc20Quest(...)` (or ERC1155).
2. **Address Calculation**: The new address = `keccak256(0xff ++ QuestFactory_address ++ nonce ++ keccak256(init_code))` — fully predictable if you know the current nonce.
3. **Funding**: The creator sends reward tokens to this new address right after deployment.
4. **Usage**: Users complete quests and claim rewards from the correct contract.

### What Actually Happens (Bug)

* No salt → address is **sequential and guessable** by anyone monitoring the chain.
* On **reorg-prone chains**:
  * Polygon PoS → frequent short reorgs (e.g., documented cases lasting 1.5+ minutes).
  * Optimism/Arbitrum (optimistic rollups) → possible longer reverts if fraud proofs are submitted.
* During a reorg, the original deployment tx can be "rewound" → nonce doesn't advance yet → attacker front-runs by deploying their own quest to the expected address.
* When the chain stabilizes, the creator's funding tx executes → tokens go to attacker's quest → attacker controls/withdraws the rewards.

### Why This Matters

* **Direct fund loss** for quest creators (projects) — stolen reward tokens.
* **Trust erosion** — projects often pre-compute addresses for multi-chain deploys or automation.
* **Realistic on L2s** — reorgs are more common/frequent than on Ethereum L1.
* **Low cost** for attacker — just gas for one deployment during reorg window.

### Concrete Walkthrough (Alice & Bob - Attacker)

* **Setup**: QuestFactory nonce = 100. Next quest address = predictable `0xabc...123`.
* **Alice creates quest**: Sends tx to call `createQuest` → pending in mempool.
* **Reorg occurs**: The block including Alice's tx is reorganized (rewound).
* **Bob (attacker) reacts fast** (script monitoring reorgs): Calls `createQuest` himself → deploys his quest to `0xabc...123` (same nonce still valid).
* **Chain stabilizes**: Alice's creation tx re-executes → but now deploys to a **new address** (nonce increased by Bob's tx).
* **Alice funds**: Sends tokens to the **original expected address** `0xabc...123` → now Bob's quest.
* **Result**: Bob controls the funded quest → can drain rewards or grief Alice.

> **Analogy**: Imagine buying a house with a predictable address based on lot number (nonce). During a city planning "reorg" (they redo the block), a squatter quickly registers the same address first. When the plans finalize, you pay for utilities/mortgage → but the money goes to the squatter's house.

## Vulnerable Code Reference

**Deployment uses standard `new` (CREATE opcode – predictable via nonce)**  

```solidity
// Around line ~75 (ERC20 case)
Erc20Quest newQuest = new Erc20Quest(
    rewardTokenAddress_,
    endTime_,
    startTime_,
    totalParticipants_,
    rewardAmountOrTokenId_,
    questId_,
    address(rabbitholeReceiptContract),
    questFee,
    protocolFeeRecipient
);

// Around line ~108 (ERC1155 case)
Erc1155Quest newQuest = new Erc1155Quest(
    rewardTokenAddress_,
    endTime_,
    startTime_,
    totalParticipants_,
    rewardAmountOrTokenId_,
    questId_,
    address(rabbitholeReceiptContract)
);
```

No CREATE2 → no salt → address only depends on factory nonce.

## Recommended Mitigation

1. **Switch to CREATE2** (primary fix) for deterministic & unique addresses.
   Use a salt that includes unique caller data (e.g., `msg.sender`, `questId_`, `rewardTokenAddress_`, chainid, timestamp) to prevent prediction/hijacking.

   Example modern implementation (post-audit fix in current repo):

   ```solidity
   // Using Solady's LibClone or similar
   address newQuest = address(new Erc20Quest).cloneDeterministic(
       salt = keccak256(abi.encodePacked(msg.sender, questId_, rewardTokenAddress_, block.chainid))
   );
   ```

2. **Add salt best practices**:
   * Include `msg.sender` → different creators get different addresses.
   * Include unique quest params → even same creator can't collide.

3. **Defensive UX** (off-chain): Warn users/projects not to rely on pre-computed addresses without waiting for finality.

4. **Tests & Invariants**: Add tests simulating reorg scenarios (hard on testnets, but possible with anvil/foundry reorg tools).

## Pattern Recognition Notes

* **Nonce-based Address Prediction**: Any factory using plain CREATE without salt is vulnerable on reorg-prone chains.
* **Reorg Windows on L2s**: Optimistic rollups & PoS sidechains have longer effective finality windows → higher risk than Ethereum L1.
* **Front-Running Deployment**: Predictable deployments are classic targets for address-squatting attacks.
* **Lack of Uniqueness Guarantee**: If address matters (e.g., funding follows immediately), must use CREATE2 + good salt.
* **Recoverability**: Hard to recover stolen funds → emphasize prevention over cure.

## Quick Recall (TL;DR)

* **Bug**: `new Quest(...)` uses CREATE → predictable address based on nonce.
* **Impact**: Reorg on Polygon/Optimism/Arbitrum → attacker hijacks address → steals quest funding/rewards.
* **Fix**: Use CREATE2 with salt including `msg.sender` + unique params (questId, token, etc.).
* **Status (Dec 2025)**: Fixed in current RabbitHole repo using deterministic cloning with proper salt.
