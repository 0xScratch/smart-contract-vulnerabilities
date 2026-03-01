# Smart Contract Vulnerabilities

This repository contains a curated list of known smart contract vulnerabilities categorized by their type. Each entry includes a brief description of the vulnerability, its severity, and a link to a detailed report or analysis.

> I try to keep adding vulnerabilities as soon as I come across one through solodit, or any other means. That said, it doesn't mean one will find every finding in here. Most of them seemed good so far.
>
> The sole purpose of this repository is just to make white hats (especially me) understand these findings in easy way possible, with the help of AI. Usually, one needs to work around a bit more in order to understand why something is a vulnerability at first place. So, this repository, decodes it in simple form and will definitely prove to useful for me, as far as I am concerned.
>
> And Most importantly, it will create a habit of me to keep reading other auditors' reports!!

## Table of Contents

- [Smart Contract Vulnerabilities](#smart-contract-vulnerabilities)
  - [Table of Contents](#table-of-contents)
  - [How each finding is listed?](#how-each-finding-is-listed)
  - [Vulnerability Categories](#vulnerability-categories)
    - [Access Control](#access-control)
    - [Business Logic Flaw](#business-logic-flaw)
    - [Denial of Service](#denial-of-service)
    - [Front Running/MEV](#front-runningmev)
    - [Governance](#governance)
    - [Insecure Randomness](#insecure-randomness)
    - [Invalid Validation](#invalid-validation)
    - [Math / Arithmetic Errors](#math--arithmetic-errors)
    - [Reentrancy](#reentrancy)
    - [Timing](#timing)
    - [Others](#others)
  - [Case Studies](#case-studies)

## How each finding is listed?

Here are the common steps I took in listing these findings:

- First, an obvious step, I went through a vulnerability (let's say from solodit, or any other means) and understood what's going on in here.
- After that the same finding is passed to AI (chatGPT or Grok), and its help is taken to understand it even better.
- Now we know our AI assistant understand it well, all I need to do is pass the template, which contains the following sections:
  - **Title**: *Self explanatory*
  - **Some extra meaningful details**
    - Severity
    - Source
    - Affected Contract
    - Vulnerability Type
  - *Some early added vulnerabilities might contain the original finding that auditors wrote*
  - **Summary**: A straight written summary about the finding
  - **A Better Explanation (With Simplified Example)**
    - **Intended Behavior**: What should happen
    - **What Actually Happens (Bug)**
    - **Why This Matters**: Impact
    - **Concrete Walkthrough**: The simplified example, it helps sometimes
  - **Vulnerable Code Reference**
  - **Recommended Mitigation**
  - **Pattern Recognition Notes**: Really Important!!
  - **Quick Recall (TL;DR)**: Only latest added findings contains this section
- Next, the generated finding by AI is usually checked and thus been added under the relevant category.
- Each category contains a table for easier accessibility.

## Vulnerability Categories

### Access Control

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Alchemix    |   [Unauthorized Reward Token Injection via `notifyRewardAmount` in Alchemix Bribe Contract](/bugs/access-control/alchemix-unauthorized-reward-token-injection-access-control-bypass.md)   |   Medium  |   Immunefi    |
|   BakerFi     |   [Arbitrary `originalAmount` in Flash Loan Data Allows Logic Manipulation](/bugs/access-control/bakerfi-flashloandata-callback-unvalidated-param.md) |   Medium  |   Code4rena   |
|   Escher  |   [Sale Finalization Failure Due to Deprecated `selfdestruct` Semantics in Escher FixedPrice & OpenEdition](/bugs/access-control/escher-sale-not-finalized-due-to-selfdestruct-semantics-change.md)   |   Medium  |   Code4rena   |
|   Karak   |   [Unslashable NativeVault via Unvalidated `extraData` and Unrestricted Manager Upgrade](/bugs/access-control/karak-nativevault-unslashable-vault-manager-upgrade.md) |   High    |   Code4rena  |
| MonoX | [Unauthorized Pool Price Manipulation via Missing Access Control in Monoswap](/bugs/access-control/monox-unauthorized-pool-price-update-access-control.md)  | High  | Solodit (Halborn) |
|   Stader  |   [Loss of Admin via Self-Assignment in `updateAdmin` (Role Revocation Bug)](/bugs/access-control/stader-staderconfig-updateadmin-same-address-access-loss.md)    |   Medium  |   Code4rena   |
|   Stader  |   [Permissionless Reward Drain Allows Unfair Operator Slashing in ValidatorWithdrawalVault](/bugs/access-control/stader-validatorwithdrawalvault-distributerewards-slashing.md)   |   Medium  |   Code4rena   |
|   Virtuals Protocol   |   [ContributionNft Mint Abuse via Unrestricted Proposer Control](/bugs/access-control/virtuals-contributionnft-mint-role-overreach.md)    |   Medium  |   Code4rena   |

---

### Business Logic Flaw

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Abracadabra |   [Improper Handling of Rebasing Tokens in Lending/Borrowing Logic](/bugs/business-logic-flaw/abracadabra-lockingmultirewards-rebasing-token-misuse.md)   |   Medium  |   Code4rena   |
| Amphora | [Reorg-Based Vault Address Hijacking via CREATE Deployment in Amphora](/bugs/business-logic-flaw/amphora-reorg-based-vault-address-hijacking-create-deployment.md)  | Medium  | Code4rena |
|   Arcade  |   [Incorrect `gscAllowance` Accounting & ERC20 Allowance Overwrite Risk in ArcadeTreasury](/bugs/business-logic-flaw/arcade-gscAllowance-erc20-allowance-overwrite.md)    |   Medium  |   Code4rena   |
|   Badger  |   [Redemption Drains Healthy CDPs → System-Wide Under-Collateralization](/bugs/business-logic-flaw/badger-redemption-healthy-cdp-drain.md)    |   Medium  |   Code4rena   |
|   Canto   |   [Epoch Boundary Reward Inflation via Misaligned `nextEpoch` Calculation in `update_market`](/bugs/business-logic-flaw/canto-epoch-reward-inflation.md)  |   High    |   Code4rena   |
| Ekubo | [TWAMM Instant Orders Can Steal Historical Rewards via Start-Time Boundary Bug](/bugs/business-logic-flaw/ekubo-twamm-instant-order-starttime-reward-inflation.md)  | Medium  | Code4rena |
|   Init Capital    |   [ReturnNative Flag Ignored in Withdraw Flow of MoneyMarketHook](/bugs/business-logic-flaw/initcapital-returnnative-ignored-withdraw.md) |   Medium  |   Code4rena   |
|   Licredity   |   [Self-Liquidation via Unlock Abuse in Licredity](/bugs/business-logic-flaw/licredity-self-liquidation-unlock-abuse.md)  |   Critical    |   Cyfrin Audits   |
|   Licredity   |   [Self-Triggered Back-Run Enables LP Fee Farming in Licredity](/bugs/business-logic-flaw/licredity-self-triggered-backrun-fee-farming.md)    |   High    |   Cyfrin Audits   |
|   Licredity   |   [Index Desynchronization via Swap-and-Pop in Position Fungibles](/bugs/business-logic-flaw/licredity-swap-pop-index-desync.md)  |   Critical    |   Cyfrin Audits   |
| Nested Finance  | [ETH Deposit Inflation via msg.value Reuse in Batched Orders](/bugs/business-logic-flaw/nested-eth-msgvalue-reuse-batched-orders.md)  | Medium  | Code4rena |
|   Notional Finance    |   [Single-Sided Redemption Slippage Bypass via Miscomputed Min Amounts](/bugs/business-logic-flaw/notional-single-sided-redemption-min-amount-miscalculation-slippage-loss.md)    |   High    |   Sherlock Audits |
|   Opensea |   [Seaport Merkle Tree Intermediate Hash Bypass in Criteria Resolution](/bugs/business-logic-flaw/opensea-seaport-criteriaresolution-merkle-intermediate-hash-bypass.md)  |   Medium  |   Code4rena   |
|   Particle    |   [Ineffective Deadline Usage in Particle's LiquidityPosition Library](/bugs/business-logic-flaw/particle-liquidityposition-deadline-flaw.md) |   Medium  |   Code4rena   |
|   PoolTogether    |   [Loss of Unclaimed Yield Fees Due to Partial Claim Reset in PrizeVault](/bugs/business-logic-flaw/pooltogether-prizevault-yield-fee-claim-accounting-bug.md)    |   High    |   Code4rena   |
|   Reserve |   [Revenue Loss via Mid-Flight Distribution Parameter Changes in Reserve Protocol Distributor](/bugs/business-logic-flaw/reserve-protocol-distributor-mid-flight-revenue-diversion.md)    |   Medium  |   Code4rena   |
|   Revert  |   [Incorrect Daily Lending/Borrowing Cap Due to Off-by-One Scaling in V3Vault](/bugs/business-logic-flaw/revert-v3vault-math-daily-limit-overflow.md) |   Medium  |   Code4rena   |
|   Sentiment   |   [Protocol Reserve Leakage via Unrestricted Borrowing in LToken Vault](/bugs/business-logic-flaw/sentiment-protocol-reserve-leakage-unrestricted-borrowing.md)   |   Medium  |   Sherlock Audits |
|   Size    |   [Incremental Compensation Blocked by Strict CR Check in `compensate()`](/bugs/business-logic-flaw/size-compensate-strict-collateral-ratio-check.md) |   Medium  |   Code4rena   |
|   Stader  |   [Consensus Stall via Strict Equality in StaderOracle Submissions](/bugs/business-logic-flaw/stader-oracle-consensus-stall.md) |   Medium  |   Code4rena|
|   Tapioca |   [Incorrect Share-to-Fraction Calculation Due to Inconsistent Rounding in MagnetarHelper](/bugs/business-logic-flaw/tapioca-withdraw-helper-rounding-mismatch.md)    |   Medium  |   Code4rena   |
|   Verwa   |   [Extra Gauge Weight via Front-Running Governance Overrides in GaugeController](/bugs/business-logic-flaw/verwa-gauge-weight-front-run-exploit.md)   |   Medium  |   Code4rena   |
|   Verwa   |   [Replay Attack in Gauge Voting via Delegation Abuse](/bugs/business-logic-flaw/verwa-gaugecontroller-voting-power-replay-via-delegation.md) |   High    |   Code4rena   |

---

### Denial of Service

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Althea Liquid   |   [Denial of Service via Empty Distribution Griefing](/bugs/denial-of-service/althea-liquid-empty-distributions-griefing-lock.md) |   Medium  |   Code4rena   |
|   Asymmetry Finance   |   [Denial of Service via Rounding Edge Case in `WstEth.withdraw`](/bugs/denial-of-service/asymmetry-finance-wsteth-withdraw-zero-unwrap.md)   |   Medium  |   Code4rena   |
|   Autonolas   |   [Global Withdraw DoS via Zero‑Liquidity Position in Liquidity Lockbox](/bugs/denial-of-service/autonolas-liquidity-lockbox-zero-liquidity.md)   |   High    |   Code4rena   |
|   Axelar  |   [Denial-of-Service via Flow-Limit Exhaustion in Axelar TokenManager](/bugs/denial-of-service/axelar-tokenmanager-flow-limit-exhaustion.md)  |   Medium  |   Code4rena   |
|   Basin   |   [Cheap DoS via Zero-Fee TWAP Manipulation in Basin](/bugs/denial-of-service/basin-well-zero-fee-slippage.md)    |   Medium  |   Code4rena   |
|   Delegate    |   [Nonce Desynchronization Leading to Denial of Service in `CreateOfferer.sol`](/bugs/denial-of-service/delegate-createofferer-nonce-desync-dos.md)   |   Medium  |   Code4rena   |
| Ekubo | [Cumulative Rounding Refund Error in TWAMM Leads to Pool-Wide DoS and Value Leakage](/bugs/denial-of-service/ekubo-twamm-rounding-error-cumulative-refund-dos.md) | Medium  | Code4rena |
|   EvmAuth |   [Incomplete Burn Handling in `_burnGroupBalances` (EVMAuthExpiringERC1155)](/bugs/denial-of-service/evmauth-incomplete-burn-groupbalances-dos.md)   |   High    |   Trail of Bits   |
|   EvmAuth |   [Incorrect Account Assignment in Token Burning Logic in EVMAuthExpiringERC1155](/bugs/denial-of-service/evmauth-incorrect-account-assignment-burn-dos.md)   |   High    |   Trail of Bits   |
|   Frankencoin |   [Fragile Challenge Finalization Due to Unchecked Transfer Failures in `end()` Function](/bugs/denial-of-service/frankencoin-mintinghub-end-unchecked-transfer-failure.md)   |   Medium  |   Code4rena   |
|   Gondi   |   [Auction DoS via Minimal Increment Bids in Gondi](/bugs/denial-of-service/gondi_auction_dos_dust_bids_time_lock.md) |   Medium  |   Code4rena   |
|   Jpegd   |   [Global Withdraw DoS via Negative-Delta Accounting in yVaultLPFarming](/bugs/denial-of-service/jpegd-yvaultlpfarming-underflow-triggered-global-dos.md) |   High    |   Code4rena   |
|   Livepeer Protocol   |   [Fully Slashed Transcoder Vote Override Denial of Service Vulnerability](/bugs/denial-of-service/livepeer-slashed-transcoder-zero-weight-vote-disruption.md)    |   Medium  |   Code4rena   |
| Nudge.xyz | [DoS on Reallocation Processing via Privileged Executor Role Revocation](/bugs/denial-of-service/nudge-dos-reallocation-executor-role-revocation.md)  | Medium  | Code4rena |
|   Phi |   [Griefing via Forced Share Lock Extension in Phi Protocol](/bugs/denial-of-service/phi-griefing-sharelock-bypass-via-buyShareCredFor.md)    |   Medium  |   Code4rena   |
|   PoolTogether    |   [Prize Tier Manipulation via Single Claim Controlling `largestTierClaimed`](/bugs/denial-of-service/pooltogether-prize-tier-manipulation.md)    |   Medium  |   Code4rena   |
|   Putty   |   [Global Withdraw DoS via Fee Transfer Revert in PuttyV2](/bugs/denial-of-service/putty-global-withdraw-dos-fee-transfer-revert.md)  |   Medium  |   Code4rena   |
|   ReNFT   |   [Rental Stop DoS via Disabled `onStop` Hook in reNFT Guard Policy](/bugs/denial-of-service/renft-onstop-hook-disable-rental-freeze.md)  |   Medium  |   Code4rena   |
|   Reserve |   [Dutch Auctions Can Fail to Settle Due to Silent Error Handling in BackingManager](/bugs/denial-of-service/reserve-rebalance-empty-revert-auction-failure.md)   |   Medium  |   Code4rena   |
|   Revolution Protocol |   [Auction Settlement DoS via Malicious Multi-Creator Setup in Revolution Protocol](/bugs/denial-of-service/revolution-auctionhouse-multicreator-dos.md)  |   Medium  |   Code4rena   |
|   Revolution Protocol |   [DoS via Gas-Intensive NFT Minting Failing AuctionHouse's Auction Creation](/bugs/denial-of-service/revolution-protocol-auctionhouse-gas-intensive-nft-minting.md)  |   Medium  |   Code4rena   |
|   Taiko   |   [Denial of Service via Permissioned Genesis Block](/bugs/denial-of-service/taiko-permissioned-genesis-block-dos.md) |   Medium  |   Code4rena   |

---

### External Call

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
| Li.Fi | [Arbitrary External Call → Token Drain via User Allowances (GenericBridgeFacet)](/bugs/external-call/lifi-arbitrary-external-call-token-drain-allowance-abuse.md) | High  | Spearbit (Solodit)  |

---

### Front Running/MEV

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Abracadabra.money | [Deterministic Pool Address Hijack via `tx.origin` on Blast (CREATE2 Collision Under Reorg)](/bugs/frontrunning-and-mev/abracadabra-deterministic-address-hijack-txorigin-blast-reorg.md)   |   Medium  |   Code4rena   |
|   Blueberry   |   [Blueberry Protocol - Disabled Deadline Enables Stale Swaps & MEV Exploitation](/bugs/frontrunning-and-mev/blueberry-disabled-swap-deadline-stale-tx-mev-exploit.md)    |   High    |   Sherlock Audits |
|   Derby Finance   |   [On-Chain Slippage Manipulation via Uniswap Quoter in Derby Finance](/bugs/frontrunning-and-mev/derbyfinance-onchain-slippage-manipulation-uniswap-quoter-mev-sandwich.md)   |   High    |   Sherlock Audits |
|   Ethereum Credit Guild |   [Auction manipulation by block stuffing and reverting on ERC-777 hooks](/bugs/frontrunning-and-mev/ethcreditguild-auction-block-stuffing.md) |   Medium  |   Code4rena   |
|   EYWA    |   [Transaction DoS via permit() Front-Running in RouterV2](/bugs/frontrunning-and-mev/eywa-permit-front-running-transaction-dos.md)    |   Medium  |   MixBytes    |
| Geode Finance | [MEV Capture of Operator Incentives via Public Arbitrage on Delayed Unstake](/bugs/frontrunning-and-mev/geodefi-mev-capture-of-operator-unstake-incentives.md)  | Medium  | Consensys (Solodit) |
|   HyperBloom  |   [Sandwich-Driven Liquidity Mint Manipulation via Calm-Period Bypass in Passive Strategy Manager](/bugs/frontrunning-and-mev/hyperbloom-calmperiod-bypass-sandwich-liquidity-mint.md)    |   Medium  |   Pashov Audit Group  |
|   InitCapital |   [Limit Price Manipulation via Front-Running Allows Theft in Margin Order Filling](/bugs/frontrunning-and-mev/initcapital-limit-price-front-run-margin-order-theft.md)   |   High    |   Code4rena   |
|   Loop Vaults |   [Incorrect Vesting Interest Calculation Enables MEV Exploitation](/bugs/frontrunning-and-mev/loop-vaults-incorrect-vesting-interest-mev.md)  |   High    |   Pashov Audit Group  |
|   Notional Finance    |   [Approval Front-Running via Allowance Overwrite in `NoteERC20`](/bugs/frontrunning-and-mev/notional-approval-front-run-allowance-overwrite.md)  |   Medium  |   OpenZeppelin Audit  |
|   Nuts Finance    |   [Initial Mint Front‑Run Inflation Attack — SelfPeggingAsset (Tapio / NUTS Finance)](/bugs/frontrunning-and-mev/nutsfinance-initial-mint-front-run-inflation-attack-selfpeggingasset.md)    |   Critical    |   Tapio Security Audit Report |
|   Rhinestone  |   [PermissionID Swap Attack via Unsigned Permission Identifier in Enable-Mode Digest](/bugs/frontrunning-and-mev/rhinestone-smartsessions-permissionid-swap-during-enable-mode.md)    |   High    |   Solodit |
|   Stealth Project |   [Front-Run Pool Initialization & Forced Mispriced Liquidity Deposit in `getOrCreatePoolAndAddLiquidity`](/bugs/frontrunning-and-mev/stealthproject-front-run-initial-tick-liquidity-mispricing.md)  |   Medium  |   Solodit (Code4rena) |
| Story Protocol (Proof of Creativity)  | [Front-Running DoS on Group IP Setup via Unauthorized License Token Minting](/bugs/frontrunning-and-mev/storyprotocol-group-ip-front-running-dos-unauthorized-mint.md)  | High  | Solodit (Halborn) |

---

### Governance

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Alchemix    |   [Zero-Supply Proposal Spam in AlchemixGovernor (Griefing Attack)](/bugs/governance/alchemix-governor-zero-supply-proposal-spam-griefing.md) |   Medium  |   Solodit (Immunefi)  |
|   Arbitrum    |   [Signature Replay in Split-Voting Governor Elections](/bugs/governance/arbitrum-signature-replay-in-governor-split-voting.md)    |   High    |   Code4rena   |
| Behodler  | [Flashloan Manipulation of LP Pricing in burnAsset & setEYEBasedAssetStake](/bugs/governance/behodler-flashloan-lp-pricing-manipulation-fate-inflation.md)  | High  | Code4rena |
|   Ethereum Credit Guild   |   [Cheap Governance Manipulation via PSM Unlimited Minting](/bugs/governance/ethereumcreditguild-psm-governance-veto.md)  |   Medium  |   Code4rena   |
|   Salty   |   [Vote Inflation via SALT Recycling in Proposals.sol](/bugs/governance/salty-ballot-vote-recycling.md)   |   Medium  |   Code4rena   |

---

### Insecure Randomness

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   AI Arena    |   [NFT Attribute Manipulation via onERC721Received Hook Revert](/bugs/insecure-randomness/ai-arena-fighter-farm-revert-to-reroll.md)  |   Medium  |   Code4rena   |

---

### Invalid Validation

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Angle   |   [Invalid Input Validation Leading to Slippage/Token Order Mismatch](/bugs/invalid-validation/angle-transmuter-redeem-token-order-slippage-mismatch.md)  |   Medium  |   Code4rena   |
| Ekubo | [Partial Fill Allowed in Single-Hop Exact-Out Swaps in Router](/bugs/invalid-validation/ekubo-router-single-hop-exact-out-partial-fill.md)  | Medium  | Code4rena |
|   Goodentry   |   [Unchecked Call Return Value in ETH Transfers](/bugs/invalid-validation/goodentry-v3proxy-unchecked-call-return-value.md)    |   Medium  |   Code4rena   |
| Illuminate  | [User-Controlled AMM Pool Mismatch Enables Theft of Protocol stETH Fees](/bugs/invalid-validation/illuminate-user-controlled-amm-pool-asset-mismatch-steth-fee-theft.md)  | High  | Sherlock  |
|   Livepeer Protocol   |   [Incorrect Vote Deduction in Livepeer Governance System](/bugs/invalid-validation/livepeer-governor-vote-miscount-delegate-bypass.md)   |   High    |   Code4rena   |
|   Maia    |   [Cross‑chain DepositNonce Poisoning — `retrieveDeposit()` allows arbitrary nonces to be marked executed](/bugs/invalid-validation/maia-branchbridgeagent-retrievedeposit-depositnonce-poisoning.md) |   High    |   Code4rena   |
|   Nextgen |   [Double Royalty Payout Due to Faulty Split Logic in NextGen Minter Contract](/bugs/invalid-validation/nextgen-minter-splits-bypass.md)  |   Medium  |   Code4rena   |
|   Olympus |   [Oracle-Based Post-Exit Skim Nullifies User Slippage Protection in `BLVaultLido`](/bugs/invalid-validation/olympus-post-exit-oracle-skim-slippage-bypass.md)    |   High    |   Sherlock Audit  |
|   Panoptic    |   [Duplicate TokenId fingerprint collision → solvency bypass in `PanopticPool.sol`](/bugs/invalid-validation/panoptic-collateral-accounting-position-duplication-insolvency-bypass.md)    |   Medium  |   Code4rena   |
|   Lindy Labs Sandlock |   [Flash-Loan Fee Ignorance Leading to Rebalance & Withdraw DoS in Sandclock Vaults](/bugs/invalid-validation/sandclock-flashloan-fees-ignored-withdraw-dos.md)   |   High    |   Solodit |
|   Venus   |   [Fragile Liquidation Check in `Comptroller.sol` — Zero Borrow Balance Requirement](/bugs/invalid-validation/venus-comptroller-liquidation-zero-balance.md)  |   Medium  |   Code4rena   |

---

### Math / Arithmetic Errors

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Isomorph    |   [Bad Debt Persistence via Truncation Mismatch in Isomorph Velo Vault](/bugs/math-error/isomorph-collateral-accounting-inflation-borrow-miscalculation.md)   |   Medium  |   Sherlock Audits |
|   Munchables  |   [Asset Freezing via Flawed Reward Penalty Calculation in LandManager](/bugs/math-error/munchables-asset-freeze-landmanager-integer-underflow.md)    |   High    |   Code4rena   |
|   OpenSea |   [Partial Order Fulfillment Discount via Low-Decimal ERC20 in `BasicOrderFulfiller`](/bugs/math-error/opensea-seaport-partial-order-fulfillment-discount-lowdecimal-erc20.md) |   Medium  |   Code4rena   |
|   Ostium  |   [Wrong Collateral Refund in Liquidation (`liqPrice == priceAfterImpact`)](/bugs/math-error/ostium-liquidation-wrong-collateral-refund.md)   |   Medium  |   Pashov Audit Group  |
|   PrePO   |   [Zero-Share Mint via Total Asset Inflation in `Collateral.sol`](/bugs/math-error/prepo-zero-shares-mint-price-manipulation-donation-attack.md)  |   High    |   Code4rena   |
|   Rigor   |   [Rounding Error Interest Loss via Day-Truncation in Interest Calculation](/bugs/math-error/rigor-interest-rounding-loss-due-to-day-truncation.md)   |   High    |   OpenCoreCh's Report |
|   Size    |   [Liquidation Profit Underflow via Decimal Mismatch in Collateral-Debt Conversion](/bugs/math-error/size-liquidation-decimal-mismatch.md)    |   High    |   Code4rena   |
|   Terplayer   |   [Withdrawal Underflow via Self-Delegation and Ceiling Division in BVT Reward Vault](/bugs/math-error/terplayer-rewardvault-withdraw-underflow-lock.md)  |   Critical    |   Shieldify Audits    |
|   Traitforge  |   [Age Underestimation Due to Early Integer Division in `calculateAge()`](/bugs/math-error/traitforge-game-mechanics-entropy-age-nuke.md) |   Medium  |   Code4rena   |

---

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
| Notional  | [Rely On Balancer Oracle Which Is Not Updated Frequently](/bugs/oracle/notional-stale-balancer-twap-oracle-price-risk-metastable2-vault.md) | Medium  | Sherlock  |

### Reentrancy

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Angle Protocol  |   [Reentrancy-Based Reward Inflation via Collateral Ratio Manipulation in Angle Transmuter](/bugs/reentrancy/angleprotocol-collateralratio-reentrancy-reward-inflation.md)    |   Medium  |   Code4rena   |
| Caviar  | [Excess ETH Theft via Royalty Recipient Reentrancy in EthRouter](/bugs/reentrancy/caviar-excess-eth-theft-via-royalty-recipient-reentrancy-ethrouter-privatepool.md)  | Medium  | Code4rena |
|   Itos    |   [Self-Transfer Settlement Bypass in `reentrantSettle`](/bugs/reentrancy/itos-reentrantsettle-selftransfer-bypass.md)    |   High    |   Pashov Audit Group  |
| Kuiper  | [Kuiper — Reentrancy-Driven Basket Drain via `settleAuction`](/bugs/reentrancy/kuiper-settleauction-reentrancy-basket-drain.md) | High  | Code4rena |
|   Panoptic    |   [Reentrancy in SemiFungiblePositionManager via ERC777 `tokensToSend` Hook](/bugs/reentrancy/panoptic-sfpm-burn-bypass-liquidity-check.md)   |   High    |   Code4rena   |
|   ReNFT   |   [Reentrancy via `safeTransferFrom` Callback in PAY Rentals](/bugs/reentrancy/renft-reclaimer-pay-rental-reentrancy.md)  |   Medium  |   Code4rena   |
|   ReNFT   |   [reNFT — ERC1155 Hijack via Reentrancy / TOCTOU (rentedAssets)](/bugs/reentrancy/renft-storage-erc1155-hijack-reentrancy.md)    |   High    |   Code4rena   |

---

### Reorg and Consensus

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
| Frankencoin | [Reorg Attack on Position Clone Creation via Predictable CREATE Address Derivation](/bugs/reorg-and-consensus/frankencoin-reorg-clone-address-prediction.md)  | Medium  | Code4rena |
| Optimism  | [Loss of Bond Amounts on Re-org Attacks in Fault Dispute Game](/bugs/reorg-and-consensus/optimism-fault-dispute-reorg-bond-loss.md) | Medium  | Sherlock Audits |
|   Optimism    |   [Incorrect DISPUTED_L2_BLOCK_NUMBER Causes Cross-Game Context Collisions & Invalid VM Outcomes](/bugs/reorg-and-consensus/optimism-uncapped-disputed-l2-block-number-cross-game-vm-context-collision.md)  |   High    |   Code4rena   |
| Rabbithole  | [Predictable Quest Address via CREATE Opcode - Reorg Attack Vulnerability](/bugs/reorg-and-consensus/rabbithole-predictable-quest-address-reorg-hijack.md)  | Medium  | Code4rena |

---


### Timing

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Basin   |   [Immutable BLOCK_TIME Parameter Cause Oracle Flaws](/bugs/timing/basin-block-time-assumption.md)    |   Medium  |   Code4rena   |
|   Frankencoin |   [Inaccurate Holding Duration on Optimism Due to `block.number` Usage in `Equity.sol`](/bugs/timing/frankencoin-equity-blocknumber-time-bug.md)    |   Medium  |   Code4rena   |
|   Karak   |   [Unfair Withdrawal Slashing During Veto Window in Karak Vaults](/bugs/timing/karak-withdraw-slash-mismatch.md)  |   Medium  |   Code4rena   |
|   Renzo   |   [L1→L2 Price Update Reverts Due to Cross-Chain Timestamp Mismatch](/bugs/timing/renzoprotocol-l1-l2-timestamp-mismatch-priceupdate-revert.md)   |   Medium    |   Code4rena   |
|   Verwa   |   [Permanent Lock via Expired-Lock Undelegation Restriction in VotingEscrow](/bugs/timing/verwa-permanent-lock-expired-undelegation.md)   |   High    |   Code4rena   |

---

### Others

|   Protocol    |   Vulnerability   |   Type    |   Severity  | Source  |
|---------------|-------------------|---------------|-----------|-----------|
|   Cap |   [Missing Slippage Protection in Liquidation Allows Unexpected Collateral Loss](/bugs/other/cap-missing-slippage-protection.md)  |   Missing Slippage Protection / Value Mismatch in Liquidation |   Medium  |   Sherlock    |
| Ekubo | [Oracle Data Corruption via Storage Key Collision in Ekubo Oracle](/bugs/other/ekubo-oracle-storage-key-collision-data-corruption.md) | Storage Collision / Data Corruption / Oracle Integrity Failure  | Medium  | Code4rena |
| NftPort | [Signature Bypass via `abi.encodePacked` Hash Collision in Factory](/bugs/other/nftport-abi-encodepacked-hash-collision-signature-bypass-factory.md)  | Authentication Bypass / Hash Collision / Input Encoding | Medium  | Sherlock  |
| Size  | [Phantom Quote Deposit via Missing Token Code Check in SizeSealed](/bugs/other/size-phantom-quote-deposit-missing-token-code-check-sizesealed.md) | Asset Validation / Phantom Deposit / Trust Assumption | Medium  | Code4rena |
|   Y2K Finance |   [EIP-4626 Interface Mismatch Causing Potential Integration Breakage in SemiFungibleVault](/bugs/other/y2kfinance-eip4626-noncompliance-integrationrisk.md)  |   Standards Non-Compliance / Composability Risk / Integration Inconsistency   |   High    |   Code4rena   |

## Case Studies

|   Protocol    |   Vulnerability   |   Type    |   Severity  | Source  |
|---------------|-------------------|---------------|-----------|-----------|
|   Curve   |   [Curve LP Oracle Manipulation via Read-Only Reentrancy](/case-studies/curve-readonly-reentrancy-oracle-manipulation.md) |   Oracle Manipulation / Read-Only Reentrancy  |   High    |   ChainSecurity   |
|   Sushi (Miso)    |   [ETH Double-Spend & Refund Exploit via BoringBatchable in MISO Auction](/case-studies/sushi_miso_eth_batch_refund_exploit.md)   |   Value Reuse / Accounting Manipulation / Refund Exploit  |   Critical    |   Samczun's blog  |
