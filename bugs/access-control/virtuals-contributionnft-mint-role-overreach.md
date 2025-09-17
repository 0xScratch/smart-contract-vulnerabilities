# ContributionNft Mint Abuse via Unrestricted Proposer Control

* **Severity**: Medium
* **Source**: [Code4rena Report](https://code4rena.com/reports/2025-04-virtuals-protocol#h-04-public-contributionnftmint-leads-to-cascading-issues--loss-of-funds)
* **Affected Contracts**:
  * [ContributionNft.sol](https://github.com/code-423n4/2025-04-virtuals-protocol/blob/28e93273daec5a9c73c438e216dde04c084be452/contracts/contribution/ContributionNft.sol)
  * [ServiceNft.sol](https://github.com/code-423n4/2025-04-virtuals-protocol/blob/28e93273daec5a9c73c438e216dde04c084be452/contracts/contribution/ServiceNft.sol)
  * [AgentRewardV2.sol](https://github.com/code-423n4/2025-04-virtuals-protocol/blob/28e93273daec5a9c73c438e216dde04c084be452/contracts/AgentRewardV2.sol)
  * [Minter.sol](https://github.com/code-423n4/2025-04-virtuals-protocol/blob/28e93273daec5a9c73c438e216dde04c084be452/contracts/token/Minter.sol)
  * [AgentDAO.sol](https://github.com/code-423n4/2025-04-virtuals-protocol/blob/28e93273daec5a9c73c438e216dde04c084be452/contracts/virtualPersona/AgentDAO.sol)
* **Vulnerability Type**: Access Control / Integrity Violation / Reward Manipulation

## Summary

The `ContributionNft::mint` function is intended to be called by the frontend after contributors submit proposals. Each proposal should generate a Contribution NFT that authenticates the submission's origin.

However, the function is **unguarded**. While it checks that the caller is the proposer of a given proposal, it allows this role to freely set arbitrary parameters (`coreId`, `newTokenURI`, `parentId`, `isModel_`, `datasetId`). This breaks downstream logic in `ServiceNft`, `AgentDAO`, `Minter`, and `AgentRewardV2`, since those components rely on the integrity of these values for impact, maturity, and reward distribution.

As a result, proposers (who already hold power to introduce proposals) can mint NFTs with **fabricated or malicious metadata**, gaining unfair rewards or damaging the system's attribution and governance logic.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. A contributor submits a proposal through the frontend.
2. The protocol mints a Contribution NFT representing this proposal.
3. Parameters such as `coreId`, `isModel`, and `datasetId` are expected to be **trustworthy**, since they influence:

   * Which service/dataset the proposal belongs to
   * Whether it represents a model or dataset
   * Which parent/child hierarchy it links into

These values are later consumed by `ServiceNft::mint`, `updateImpact`, and reward/maturity calculations.

### What Actually Happens (Bug)

* `ContributionNft::mint` allows the **proposer** to directly provide values for `coreId`, `isModel_`, `datasetId`, etc.
* There are **no validation checks** on these fields.
* Therefore, a proposer can intentionally or mistakenly set:

  * Wrong `coreId` → mis-assigns contributions to another core service.
  * Wrong `isModel_` → dataset minted as model (or vice versa).
  * Wrong `datasetId` → overwrites or hijacks someone else's dataset.

These corrupted values propagate downstream:

* **ServiceNft**: Stores incorrect core and model links.
* **updateImpact**: Miscalculates `_impacts` and dataset scores.
* **AgentRewardV2**: Distributes rewards based on bad `impact`.
* **Minter**: Transfers excess/insufficient funds.
* **AgentDAO**: Calculates incorrect maturity from fake cores.

### Why This Matters

* **Financial impact**: Rewards (tokens) are distributed based on manipulated impact → unfair payouts.
* **Governance impact**: Incorrect maturity and core assignments distort decision-making.
* **Data integrity**: Parent/child contribution graph polluted with bogus NFTs.
* **System trust**: Attackers gain influence without real contributions.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice submits a genuine dataset proposal.

* **Mallory (proposer role)**:

  * Calls `ContributionNft::mint` with:

    * `coreId = Alice's coreId`
    * `datasetId = Alice's datasetId`
    * `isModel_ = true` (though it's actually a dataset)
  * Mallory's NFT now looks like it belongs to Alice's dataset.

* **Impact propagation**:

  * `ServiceNft::mint` stores Mallory's NFT under Alice's dataset.
  * `updateImpact` miscalculates dataset score, diverting value.
  * `AgentRewardV2::_distributeContributorRewards` gives Mallory undeserved rewards.
  * `Minter::mint` mints excess tokens.

👉 Result: Mallory siphons rewards and distorts dataset attribution, while Alice (and others) get underpaid.

## Vulnerable Code Reference

**ContributionNft.sol** (unguarded mint parameters)

```solidity
function mint(
    address to,
    uint256 virtualId,
    uint8 coreId,
    string memory newTokenURI,
    uint256 proposalId,
    uint256 parentId,
    bool isModel_,
    uint256 datasetId
) external returns (uint256) {
    IGovernor personaDAO = getAgentDAO(virtualId);

    require(
        msg.sender == personaDAO.proposalProposer(proposalId),
        "Only proposal proposer can mint Contribution NFT"
    );

    require(parentId != proposalId, "Cannot be parent of itself");
    // ⚠️ Missing validation on coreId, datasetId, isModel_, etc.
}
```

### ServiceNft.sol (uses corrupted values)

```solidity
_cores[proposalId] = IContributionNft(contributionNft).getCore(proposalId); // incorrect core
bool isModel = IContributionNft(contributionNft).isModel(proposalId);       // incorrect model
```

### updateImpact()

```solidity
uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId); // hijacked datasetId
_impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;                // wrong impact
```

### AgentRewardV2::\_distributeContributorRewards

```solidity
impact = serviceNftContract.getImpact(serviceId); // manipulated impact
```

## Recommended Mitigation

1. **Restrict minting authority**
   Only allow `admin` or a trusted operator (not the proposer) to mint NFTs:

    ```solidity
    require(_msgSender() == _admin, "Only admin can mint tokens");
    ```

2. **Validate critical parameters**

   * Ensure `coreId` maps to a valid existing core.
   * Ensure `datasetId` references an owned dataset.
   * Enforce `isModel` consistency with proposal type.

3. **Decouple proposal submission from NFT minting**

   * Proposer only submits proposal.
   * DAO/backend verifies and calls `mint` with validated parameters.

4. **Add invariant checks**

   * A proposal cannot overwrite someone else's dataset.
   * Parent/child IDs cannot form cycles or self-links.

## Pattern Recognition Notes

* **Role Overreach**: Proposers should not directly mint NFTs with unchecked metadata. This blurs the line between *suggesting* and *authorizing*.
* **Unchecked Input Propagation**: Bad inputs ripple through multiple modules, multiplying impact.
* **Reward Manipulation via Metadata**: When payouts depend on unverified fields, malicious users can self-allocate rewards.
* **Integrity vs. Access Control**: Even trusted roles (like proposers) must not bypass validation; otherwise, the system relies on goodwill rather than code.

### Quick Recall (TL;DR)

* **Bug**: Proposers can mint Contribution NFTs with arbitrary parameters.
* **Impact**: Manipulated `coreId`, `isModel`, or `datasetId` break service attribution, impact scores, maturity, and reward distribution.
* **Fix**: Restrict `mint` to admin/operator, validate parameters, and enforce consistency with verified proposals.
