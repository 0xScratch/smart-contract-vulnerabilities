# Smart Contract Vulnerabilities

## Access Control

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   BakerFi     |   [Arbitrary `originalAmount` in Flash Loan Data Allows Logic Manipulation](/bugs/access-control/bakerfi-flashloandata-callback-unvalidated-param.md) |   Medium  |   Code4rena   |
|   Karak   |   [Unslashable NativeVault via Unvalidated `extraData` and Unrestricted Manager Upgrade](/bugs/access-control/karak-nativevault-unslashable-vault-manager-upgrade.md) |   High    |   Code4rena  |
|   Stader  |   [Loss of Admin via Self-Assignment in `updateAdmin` (Role Revocation Bug)](/bugs/access-control/stader-staderconfig-updateadmin-same-address-access-loss.md)    |   Medium  |   Code4rena   |
|   Stader  |   [Permissionless Reward Drain Allows Unfair Operator Slashing in ValidatorWithdrawalVault](/bugs/access-control/stader-validatorwithdrawalvault-distributerewards-slashing.md)   |   Medium  |   Code4rena   |
|   Virtuals Protocol   |   [ContributionNft Mint Abuse via Unrestricted Proposer Control](/bugs/access-control/virtuals-contributionnft-mint-role-overreach.md)    |   Medium  |   Code4rena   |

## Business Logic Flaw

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Abracadabra |   [Improper Handling of Rebasing Tokens in Lending/Borrowing Logic](/bugs/business-logic-flaw/abracadabra-lockingmultirewards-rebasing-token-misuse.md)   |   Medium  |   Code4rena   |
|   Arcade  |   [Incorrect `gscAllowance` Accounting & ERC20 Allowance Overwrite Risk in ArcadeTreasury](/bugs/business-logic-flaw/arcade-gscAllowance-erc20-allowance-overwrite.md)    |   Medium  |   Code4rena   |
|   Badger  |   [Redemption Drains Healthy CDPs → System-Wide Under-Collateralization](/bugs/business-logic-flaw/badger-redemption-healthy-cdp-drain.md)    |   Medium  |   Code4rena   |
|   Canto   |   [Epoch Boundary Reward Inflation via Misaligned `nextEpoch` Calculation in `update_market`](/bugs/business-logic-flaw/canto-epoch-reward-inflation.md)  |   High    |   Code4rena   |
|   Init Capital    |   [ReturnNative Flag Ignored in Withdraw Flow of MoneyMarketHook](/bugs/business-logic-flaw/initcapital-returnnative-ignored-withdraw.md) |   Medium  |   Code4rena   |
|   Licredity   |   [Self-Liquidation via Unlock Abuse in Licredity](/bugs/business-logic-flaw/licredity-self-liquidation-unlock-abuse.md)  |   Critical    |   Cyfrin Audits   |
|   Licredity   |   [Self-Triggered Back-Run Enables LP Fee Farming in Licredity](/bugs/business-logic-flaw/licredity-self-triggered-backrun-fee-farming.md)    |   High    |   Cyfrin Audits   |
|   Licredity   |   [Index Desynchronization via Swap-and-Pop in Position Fungibles](/bugs/business-logic-flaw/licredity-swap-pop-index-desync.md)  |   Critical    |   Cyfrin Audits   |
|   Particle    |   [Ineffective Deadline Usage in Particle's LiquidityPosition Library](/bugs/business-logic-flaw/particle-liquidityposition-deadline-flaw.md) |   Medium  |   Code4rena   |
|   PoolTogether    |   [Loss of Unclaimed Yield Fees Due to Partial Claim Reset in PrizeVault](/bugs/business-logic-flaw/pooltogether-prizevault-yield-fee-claim-accounting-bug.md)    |   High    |   Code4rena   |
|   Reserve |   [Revenue Loss via Mid-Flight Distribution Parameter Changes in Reserve Protocol Distributor](/bugs/business-logic-flaw/reserve-protocol-distributor-mid-flight-revenue-diversion.md)    |   Medium  |   Code4rena   |
|   Revert  |   [Incorrect Daily Lending/Borrowing Cap Due to Off-by-One Scaling in V3Vault](/bugs/business-logic-flaw/revert-v3vault-math-daily-limit-overflow.md) |   Medium  |   Code4rena   |
|   Size    |   [Incremental Compensation Blocked by Strict CR Check in `compensate()`](/bugs/business-logic-flaw/size-compensate-strict-collateral-ratio-check.md) |   Medium  |   Code4rena   |
|   Tapioca |   [Incorrect Share-to-Fraction Calculation Due to Inconsistent Rounding in MagnetarHelper](/bugs/business-logic-flaw/tapioca-withdraw-helper-rounding-mismatch.md)    |   Medium  |   Code4rena   |
|   Verwa   |   [Extra Gauge Weight via Front-Running Governance Overrides in GaugeController](/bugs/business-logic-flaw/verwa-gauge-weight-front-run-exploit.md)   |   Medium  |   Code4rena   |
|   Verwa   |   [Replay Attack in Gauge Voting via Delegation Abuse](/bugs/business-logic-flaw/verwa-gaugecontroller-voting-power-replay-via-delegation.md) |   High    |   Code4rena   |

## Denial of Service

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Althea Liquid   |   [Denial of Service via Empty Distribution Griefing](/bugs/denial-of-service/althea-liquid-empty-distributions-griefing-lock.md) |   Medium  |   Code4rena   |
|   Asymmetry Finance   |   [Denial of Service via Rounding Edge Case in `WstEth.withdraw`](/bugs/denial-of-service/asymmetry-finance-wsteth-withdraw-zero-unwrap.md)   |   Medium  |   Code4rena   |
|   Autonolas   |   [Global Withdraw DoS via Zero‑Liquidity Position in Liquidity Lockbox](/bugs/denial-of-service/autonolas-liquidity-lockbox-zero-liquidity.md)   |   High    |   Code4rena   |
|   Axelar  |   [Denial-of-Service via Flow-Limit Exhaustion in Axelar TokenManager](/bugs/denial-of-service/axelar-tokenmanager-flow-limit-exhaustion.md)  |   Medium  |   Code4rena   |
|   Basin   |   [Cheap DoS via Zero-Fee TWAP Manipulation in Basin](/bugs/denial-of-service/basin-well-zero-fee-slippage.md)    |   Medium  |   Code4rena   |
|   Delegate    |   [Nonce Desynchronization Leading to Denial of Service in `CreateOfferer.sol`](/bugs/denial-of-service/delegate-createofferer-nonce-desync-dos.md)   |   Medium  |   Code4rena   |
|   EvmAuth |   [Incomplete Burn Handling in `_burnGroupBalances` (EVMAuthExpiringERC1155)](/bugs/denial-of-service/evmauth-incomplete-burn-groupbalances-dos.md)   |   High    |   Trail of Bits   |
|   EvmAuth |   [Incorrect Account Assignment in Token Burning Logic in EVMAuthExpiringERC1155](/bugs/denial-of-service/evmauth-incorrect-account-assignment-burn-dos.md)   |   High    |   Trail of Bits   |
|   Frankencoin |   [Fragile Challenge Finalization Due to Unchecked Transfer Failures in `end()` Function](/bugs/denial-of-service/frankencoin-mintinghub-end-unchecked-transfer-failure.md)   |   Medium  |   Code4rena   |
|   Gondi   |   [Auction DoS via Minimal Increment Bids in Gondi](/bugs/denial-of-service/gondi_auction_dos_dust_bids_time_lock.md) |   Medium  |   Code4rena   |
|   Livepeer Protocol   |   [Fully Slashed Transcoder Vote Override Denial of Service Vulnerability](/bugs/denial-of-service/livepeer-slashed-transcoder-zero-weight-vote-disruption.md)    |   Medium  |   Code4rena   |
|   Phi |   [Griefing via Forced Share Lock Extension in Phi Protocol](/bugs/denial-of-service/phi-griefing-sharelock-bypass-via-buyShareCredFor.md)    |   Medium  |   Code4rena   |
|   PoolTogether    |   [Prize Tier Manipulation via Single Claim Controlling `largestTierClaimed`](/bugs/denial-of-service/pooltogether-prize-tier-manipulation.md)    |   Medium  |   Code4rena   |
|   ReNFT   |   [Rental Stop DoS via Disabled `onStop` Hook in reNFT Guard Policy](/bugs/denial-of-service/renft-onstop-hook-disable-rental-freeze.md)  |   Medium  |   Code4rena   |
|   Reserve |   [Dutch Auctions Can Fail to Settle Due to Silent Error Handling in BackingManager](/bugs/denial-of-service/reserve-rebalance-empty-revert-auction-failure.md)   |   Medium  |   Code4rena   |
|   Revolution Protocol |   [Auction Settlement DoS via Malicious Multi-Creator Setup in Revolution Protocol](/bugs/denial-of-service/revolution-auctionhouse-multicreator-dos.md)  |   Medium  |   Code4rena   |
|   Revolution Protocol |   [DoS via Gas-Intensive NFT Minting Failing AuctionHouse's Auction Creation](/bugs/denial-of-service/revolution-protocol-auctionhouse-gas-intensive-nft-minting.md)  |   Medium  |   Code4rena   |
|   Taiko   |   [Denial of Service via Permissioned Genesis Block](/bugs/denial-of-service/taiko-permissioned-genesis-block-dos.md) |   Medium  |   Code4rena   |

## Front Running/MEV

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Nuts Finance    |   [Initial Mint Front‑Run Inflation Attack — SelfPeggingAsset (Tapio / NUTS Finance)](/bugs/front-running/nutsfinance-initial-mint-front-run-inflation-attack-selfpeggingasset.md)    |   Critical    |   Tapio Security Audit Report |
|   Ethereum Credit Guild |   [Auction manipulation by block stuffing and reverting on ERC-777 hooks](/bugs/mev/ethcreditguild-auction-block-stuffing.md) |   Medium  |   Code4rena   |
|   Loop Vaults |   [Incorrect Vesting Interest Calculation Enables MEV Exploitation](/bugs/mev/loop-vaults-incorrect-vesting-interest-mev.md)  |   High    |   Pashov Audit Group  |

## Governance

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Ethereum Credit Guild   |   [Cheap Governance Manipulation via PSM Unlimited Minting](/bugs/governance/ethereumcreditguild-psm-governance-veto.md)  |   Medium  |   Code4rena   |
|   Salty   |   [Vote Inflation via SALT Recycling in Proposals.sol](/bugs/governance/salty-ballot-vote-recycling.md)   |   Medium  |   Code4rena   |

## Insecure Randomness

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   AI Arena    |   [NFT Attribute Manipulation via onERC721Received Hook Revert](/bugs/insecure-randomness/ai-arena-fighter-farm-revert-to-reroll.md)  |   Medium  |   Code4rena   |

## Invalid Validation

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Angle   |   [Invalid Input Validation Leading to Slippage/Token Order Mismatch](/bugs/invalid-validation/angle-transmuter-redeem-token-order-slippage-mismatch.md)  |   Medium  |   Code4rena   |
|   Livepeer Protocol   |   [Incorrect Vote Deduction in Livepeer Governance System](/bugs/invalid-validation/livepeer-governor-vote-miscount-delegate-bypass.md)   |   High    |   Code4rena   |
|   Maia    |   [Cross‑chain DepositNonce Poisoning — `retrieveDeposit()` allows arbitrary nonces to be marked executed](/bugs/invalid-validation/maia-branchbridgeagent-retrievedeposit-depositnonce-poisoning.md) |   High    |   Code4rena   |
|   Nextgen |   [Double Royalty Payout Due to Faulty Split Logic in NextGen Minter Contract](/bugs/invalid-validation/nextgen-minter-splits-bypass.md)  |   Medium  |   Code4rena   |
|   Panoptic    |   [Duplicate TokenId fingerprint collision → solvency bypass in `PanopticPool.sol`](/bugs/invalid-validation/panoptic-collateral-accounting-position-duplication-insolvency-bypass.md)    |   Medium  |   Code4rena   |
|   Venus   |   [Fragile Liquidation Check in `Comptroller.sol` — Zero Borrow Balance Requirement](/bugs/invalid-validation/venus-comptroller-liquidation-zero-balance.md)  |   Medium  |   Code4rena   |

## Math / Arithmetic Errors

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Munchables  |   [Asset Freezing via Flawed Reward Penalty Calculation in LandManager](/bugs/math-error/munchables-asset-freeze-landmanager-integer-underflow.md)    |   High    |   Code4rena   |
|   Ostium  |   [Wrong Collateral Refund in Liquidation (`liqPrice == priceAfterImpact`)](/bugs/math-error/ostium-liquidation-wrong-collateral-refund.md)   |   Medium  |   Pashov Audit Group  |
|   Size    |   [Liquidation Profit Underflow via Decimal Mismatch in Collateral-Debt Conversion](/bugs/math-error/size-liquidation-decimal-mismatch.md)    |   High    |   Code4rena   |
|   Terplayer   |   [Withdrawal Underflow via Self-Delegation and Ceiling Division in BVT Reward Vault](/bugs/math-error/terplayer-rewardvault-withdraw-underflow-lock.md)  |   Critical    |   Shieldify Audits    |
|   Traitforge  |   [Age Underestimation Due to Early Integer Division in `calculateAge()`](/bugs/math-error/traitforge-game-mechanics-entropy-age-nuke.md) |   Medium  |   Code4rena   |

## Reentrancy

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Itos    |   [Self-Transfer Settlement Bypass in `reentrantSettle`](/bugs/reentrancy/itos-reentrantsettle-selftransfer-bypass.md)    |   High    |   Pashov Audit Group  |
|   Panoptic    |   [Reentrancy in SemiFungiblePositionManager via ERC777 `tokensToSend` Hook](/bugs/reentrancy/panoptic-sfpm-burn-bypass-liquidity-check.md)   |   High    |   Code4rena   |
|   ReNFT   |   [Reentrancy via `safeTransferFrom` Callback in PAY Rentals](/bugs/reentrancy/renft-reclaimer-pay-rental-reentrancy.md)  |   Medium  |   Code4rena   |
|   ReNFT   |   [reNFT — ERC1155 Hijack via Reentrancy / TOCTOU (rentedAssets)](/bugs/reentrancy/renft-storage-erc1155-hijack-reentrancy.md)    |   High    |   Code4rena   |

## Timing

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   Basin   |   [Immutable BLOCK_TIME Parameter Cause Oracle Flaws](/bugs/timing/basin-block-time-assumption.md)    |   Medium  |   Code4rena   |
|   Frankencoin |   [Inaccurate Holding Duration on Optimism Due to `block.number` Usage in `Equity.sol`](/bugs/timing/frankencoin-equity-blocknumber-time-bug.md)    |   Medium  |   Code4rena   |
|   Karak   |   [Unfair Withdrawal Slashing During Veto Window in Karak Vaults](/bugs/timing/karak-withdraw-slash-mismatch.md)  |   Medium  |   Code4rena   |
|   Verwa   |   [Permanent Lock via Expired-Lock Undelegation Restriction in VotingEscrow](/bugs/timing/verwa-permanent-lock-expired-undelegation.md)   |   High    |   Code4rena   |

## Others

|   Protocol    |   Vulnerability   |   Type    |   Severity  | Source  |
|---------------|-------------------|---------------|-----------|-----------|
|   Arbitrum    |   [Signature Replay in Split-Voting Governor Elections](/bugs/other/arbitrum-signature-replay-in-governor-split-voting.md)    |   Signature Replay / Missing Nonce / Authorization Bypass |   High    |   Code4rena   |
|   Cap |   [Missing Slippage Protection in Liquidation Allows Unexpected Collateral Loss](/bugs/other/cap-missing-slippage-protection.md)  |   Missing Slippage Protection / Value Mismatch in Liquidation |   Medium  |   Sherlock    |
|   Goodentry   |   [Unchecked Call Return Value in ETH Transfers](/bugs/other/goodentry-v3proxy-unchecked-call-return-value.md)    |   call/delegatecall - Unchecked Return Value  |   Medium  |   Code4rena   |
|   Stader  |   [Consensus Stall via Strict Equality in StaderOracle Submissions](/bugs/other/stader-oracle-consensus-stall.md) |   Logic Error / Consensus Liveness Failure    |   Medium  |   Code4rena|

## Case Studies

|   Protocol    |   Vulnerability   |   Type    |   Severity  | Source  |
|---------------|-------------------|---------------|-----------|-----------|
|   Curve   |   [Curve LP Oracle Manipulation via Read-Only Reentrancy](/case-studies/curve-readonly-reentrancy-oracle-manipulation.md) |   Oracle Manipulation / Read-Only Reentrancy  |   High    |   ChainSecurity   |
|   Sushi (Miso)    |   [ETH Double-Spend & Refund Exploit via BoringBatchable in MISO Auction](/case-studies/sushi_miso_eth_batch_refund_exploit.md)   |   Value Reuse / Accounting Manipulation / Refund Exploit  |   Critical    |   Samczun's blog  |
