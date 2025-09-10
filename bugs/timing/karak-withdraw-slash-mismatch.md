# Unfair Withdrawal Slashing During Veto Window in Karak Vaults

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-07-karak-findings/issues/17) / [One Bug Per Day](https://www.onebugperday.com/v1/1408)
* **Affected Contracts**:

  * [`Constants.sol`](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/interfaces/Constants.sol)
  * [`SlasherLib.sol`](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol)
  * [`Vault.sol`](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Vault.sol)
  * [`VaultLib.sol`](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/VaultLib.sol)
  * [`Core.sol`](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Core.sol)
* **Vulnerability Type**: Timing / Accounting Mismatch / Withdrawal Safety Bypass

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/interfaces/Constants.sol#L13-L16](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/interfaces/Constants.sol#L13-L16)
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L94-L124](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L94-L124)
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L79-L92](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L79-L92)
>[https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Vault.sol#L157-L188](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Vault.sol#L157-L188)
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/VaultLib.sol#L24-L38](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/VaultLib.sol#L24-L38)
>[https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Core.sol#L248-L256](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Core.sol#L248-L256)
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L126-L151](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L126-L151)
>
>## Vulnerability details
>
>## Impact
>
>According to [https://github.com/code-423n4/2024-07-karak?tab=readme-ov-file#stakers](https://github.com/code-423n4/2024-07-karak?tab=readme-ov-file#stakers), `Stakers can initiate a withdrawal, subject to a ``MIN_WITHDRAW_DELAY`` of 9 days`, and `DSS can slash any malicious behavior occurring before the withdrawal initiation for up to 7 days`. Since `MIN_WITHDRAW_DELAY` equals `SLASHING_WINDOW + SLASHING_VETO_WINDOW`, the staker's withdrawal should be safe without being associated with any malicious behavior and hence not slashable when the DSS does not request any slashing against the corresponding vault within the `SLASHING_WINDOW` of 7 days after such withdrawal is initiated.
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/interfaces/Constants.sol#L13-L16](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/interfaces/Constants.sol#L13-L16)
>
>```solidity
>    uint256 public constant SLASHING_WINDOW = 7 days;
>    uint256 public constant SLASHING_VETO_WINDOW = 2 days;
>    uint256 public constant MIN_STAKE_UPDATE_DELAY = SLASHING_WINDOW + SLASHING_VETO_WINDOW;
>    uint256 public constant MIN_WITHDRAWAL_DELAY = SLASHING_WINDOW + SLASHING_VETO_WINDOW;
>```
>
>During the 2 day period after the `SLASHING_WINDOW` of 7 days is passed after the staker's withdrawal is initiated, such as when just couple minutes are passed after such `SLASHING_WINDOW` is reached, the staker's withdrawal cannot be finished but a malicious behavior can occur and the DSS can make a slashing request against the corresponding vault; in this situation, the staker's withdrawal mentioned previously should not be subject to such slashing though. At that time, when the DSS calls the `Core.requestSlashing` function, which further calls the `SlasherLib.requestSlashing` and `SlasherLib.fetchEarmarkedStakes` functions, the token amount to be slashed is calculated as the DSS's allowed slashing percentage of the vault's `totalAssets()`, which includes the token amount corresponding to the staker's withdrawal. Because the staker's withdrawal should not be slashable, the token amount to be slashed actually should be the DSS's allowed slashing percentage of a value that equals the vault's `totalAssets()` minus the token amount corresponding to the staker's withdrawal. Thus, the DSS can unfairly slash more underlying token amount from the vault than it should be allowed.
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L94-L124](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L94-L124)
>
>```solidity
>    function requestSlashing(
>        CoreLib.Storage storage self,
>        IDSS dss,
>        SlashRequest memory slashingMetadata,
>        uint256 nonce
>    ) external returns (QueuedSlashing memory queuedSlashing) {
>        validateRequestSlashingParams(self, slashingMetadata, dss);
>        uint256[] memory earmarkedStakes = fetchEarmarkedStakes(slashingMetadata);
>        queuedSlashing = QueuedSlashing({
>            dss: dss,
>            timestamp: uint96(block.timestamp),
>            operator: slashingMetadata.operator,
>            vaults: slashingMetadata.vaults,
>            earmarkedStakes: earmarkedStakes,
>            nonce: nonce
>        });
>        self.slashingRequests[calculateRoot(queuedSlashing)] = true;
>        ...
>    }
>```
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L79-L92](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L79-L92)
>
>```solidity
>    function fetchEarmarkedStakes(SlashRequest memory slashingMetadata)
>        internal
>        view
>        returns (uint256[] memory earmarkedStakes)
>    {
>        earmarkedStakes = new uint256[](slashingMetadata.vaults.length);
>        for (uint256 i = 0; i < slashingMetadata.vaults.length; ++i) {
>            earmarkedStakes[i] = Math.mulDiv(
>                slashingMetadata.slashPercentagesWad[i],
>                IKarakBaseVault(slashingMetadata.vaults[i]).totalAssets(),
>                Constants.MAX_SLASHING_PERCENT_WAD
>            );
>        }
>    }
>```
>
>Moreover, when the `MIN_WITHDRAW_DELAY` of 9 days is passed after the staker's withdrawal is initiated, the staker can call the `Vault.finishRedeem` function for finishing his withdrawal request. Since `SLASHING_VETO_WINDOW` is 2 days, the DSS can call the `Core.finalizeSlashing` function for finalizing its slashing request just after the `MIN_WITHDRAW_DELAY` of 9 days is passed after the initiation of the staker's withdrawal if the DSS's slashing request was made when just couple minutes were passed after the `SLASHING_WINDOW` for such withdrawal of the staker was reached. Because both the staker's `Vault.finishRedeem` transaction and the DSS's `Core.finalizeSlashing` transaction are sent at the similar time, a malicious miner can place and execute the DSS's `Core.finalizeSlashing` transaction before the staker's `Vault.finishRedeem` transaction. In this case, the staker's withdrawal can be unfairly slashed by the DSS even though it should not be slashed.
>
>[https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Vault.sol#L157-L188](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Vault.sol#L157-L188)
>
>```solidity
>    function finishRedeem(bytes32 withdrawalKey)
>        external
>        nonReentrant
>        whenFunctionNotPaused(Constants.PAUSE_VAULT_FINISH_REDEEM)
>    {
>        (VaultLib.State storage state, VaultLib.Config storage config) = _storage();
>
>        WithdrawLib.QueuedWithdrawal memory startedWithdrawal = state.validateQueuedWithdrawal(withdrawalKey);
>        ...
>    }
>```
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/VaultLib.sol#L24-L38](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/VaultLib.sol#L24-L38)
>
>```solidity
>    function validateQueuedWithdrawal(State storage self, bytes32 withdrawalKey)
>        internal
>        view
>        returns (WithdrawLib.QueuedWithdrawal memory qdWithdrawal)
>    {
>        qdWithdrawal = self.withdrawalMap[withdrawalKey];
>        ...
>        if (qdWithdrawal.start + Constants.MIN_WITHDRAWAL_DELAY > block.timestamp) {
>            revert MinWithdrawDelayNotPassed();
>        }
>    }
>```
>
>[https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Core.sol#L248-L256](https://github.com/code-423n4/2024-07-karak/blob/53eb78ebda718d752023db4faff4ab1567327db4/src/Core.sol#L248-L256)
>
>```solidity
>    function finalizeSlashing(SlasherLib.QueuedSlashing memory queuedSlashing)
>        external
>        nonReentrant
>        whenFunctionNotPaused(Constants.PAUSE_CORE_FINALIZE_SLASHING)
>    {
>        _self().finalizeSlashing(queuedSlashing);
>        ...
>    }
>```
>
>[https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L126-L151](https://github.com/code-423n4/2024-07-karak/blob/d19a4de35bcaf31ccec8bccd36e2d26594d05aad/src/entities/SlasherLib.sol#L126-L151)
>
>```solidity
>    function finalizeSlashing(CoreLib.Storage storage self, QueuedSlashing memory queuedSlashing) external {
>        ...
>        if (queuedSlashing.timestamp + Constants.SLASHING_VETO_WINDOW > block.timestamp) {
>            revert MinSlashingDelayNotPassed();
>        }
>        ...
>    }
>```
>
>## Proof of Concept
>
>The following steps can occur for the first described scenario where the DSS unfairly slashes more underlying token amount from the vault than it should be allowed.
>
>1. The vault holds `100` underlying tokens and has only one staker who owns `100` shares.
>2. On Day 0, the staker initiates his withdrawal for redeeming `50` shares.
>3. During the `SLASHING_WINDOW` of 7 days between Day 0 and Day 7, no malicious behavior occurs, and the DSS does not request any slashing against the vault.
>     * Therefore, the staker's withdrawal is safe and should not be slashable, and the staker should receive `50 * 100 / 100 = 50` underlying tokens for his withdrawal.
>4. On Day 8, a malicious behavior occurs, and the DSS with a 50% allowed slashing percentage requests to slash the vault.
>     * The token amount to be slashed by the DSS is calculated to be `50% * 100 = 50` underlying tokens.
>     * However, because the staker's withdrawal initiated on Day 0 should not be subject to such slashing, the token amount to be slashed by the DSS should equal `50% * (100 - 50) = 25` underlying tokens.
>5. On Day 9, the MIN_WITHDRAW_DELAY of 9 days is passed after the initiation of the staker's withdrawal so the staker finishes his withdrawal and does receive `50 * 100 / 100 = 50` underlying tokens from the vault.
>6. On Day 10, the DSS finalizes the slashing and slashes `50` underlying tokens calculated on Day 8, which is more than `25` underlying tokens that it should slash, from the vault. Therefore, the DSS unfairly slashes more underlying tokens from the vault than it should be allowed.
>
>The following steps can occur for the second described scenario where the staker's withdrawal can be unfairly slashed by the DSS even though it should not be slashed.
>
>1. The vault holds `150` underlying tokens and has two stakers in which Staker A owns `100` shares and Staker B owns `50` shares.
>2. On Day 0, Staker A initiates his withdrawal for redeeming `100` shares.
>3. During the `SLASHING_WINDOW` of 7 days between Day 0 and Day 7, no malicious behavior occurs, and the DSS does not request any slashing against the vault.
>     * Therefore, Staker A's withdrawal is safe and should not be slashable, and Staker A should receive `100 * 150 / (100 + 50) = 100` underlying tokens for his withdrawal.
>4. A malicious behavior occurs when 3 minutes are passed after Day 7 starts, and the DSS with a 50% allowed slashing percentage requests to slash the vault when 5 minutes are passed after Day 7 starts.
>     * The token amount to be slashed by the DSS is calculated to be `50% * 150 = 75` underlying tokens.
>     * However, because Staker A's withdrawal initiated on Day 0 should not be subject to such slashing, the token amount to be slashed by the DSS should equal `50% * (150 - 100) = 25` underlying tokens.
>5. On Day 9, the `MIN_WITHDRAW_DELAY` of 9 days is passed after the initiation of Staker A's withdrawal so the staker calls the `Vault.finishRedeem` function for finishing his withdrawal.
>6. When 5 minutes are passed after Day 9 starts, the DSS calls the `Core.finalizeSlashing` function for finalizing the slashing.
>7. Both Staker A's `Vault.finishRedeem` transaction and the DSS's `Core.finalizeSlashing` transaction are sent at the similar time, a malicious miner places and executes the DSS's `Core.finalizeSlashing` transaction before Staker A's `Vault.finishRedeem` transaction.
>8. When the DSS's `Core.finalizeSlashing` transaction is executed, the DSS slashes `75` underlying tokens calculated in Step 4, which is more than `25` underlying tokens that it should slash, from the vault.
>9. When Staker A's `Vault.finishRedeem` transaction is executed, Staker A receives `100 * (150 - 75) / (100 + 50) = 50` underlying tokens, which is less than `100` underlying tokens that it should receive for his withdrawal, from the vault. Therefore, Staker A's withdrawal is unfairly slashed by the DSS even though it should not be slashed.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>One way to mitigate this issue is to update the `Vault.finishRedeem` function to allow the staker's withdrawal to be finished after the `SLASHING_WINDOW` of 7 days is passed after such withdrawal is initiated if the DSS does not request any slashing against the corresponding vault within such `SLASHING_WINDOW` and only enforce the `MIN_WITHDRAW_DELAY` of 9 days on the withdrawal if the DSS has requested to slash the corresponding vault within such `SLASHING_WINDOW`.
>
>## Assessed type
>
>Timing

## Summary

Karak vault withdrawals are designed to be safe once the **7-day slashing window** passes. However, because `Vault.totalAssets()` still includes pending withdrawals until **9 days** (MIN\_WITHDRAW\_DELAY), DSS can **request and finalize slashing during the 2-day veto window (days 7-9)** that unfairly counts "safe" withdrawals as slashable.

This allows:

1. **Over-slashing**: More tokens are removed from the vault than should be allowed.
2. **Withdrawal theft**: Stakers who survived the slashing window still lose part (or all) of their withdrawal if a miner prioritizes DSS's `finalizeSlashing()` over the staker's `finishRedeem()`.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* **Withdraw initiation**: Locked for 9 days (`SLASHING_WINDOW + SLASHING_VETO_WINDOW`).
* **Days 0-7**: Withdrawal slashable if DSS reports misbehavior.
* **After day 7**: If no DSS slashing request → withdrawal is safe, should not be included in future slashing.
* **Day 9+**: Staker calls `finishRedeem()` to receive tokens.

### What Actually Happens (Bug)

* `SlasherLib.fetchEarmarkedStakes` calculates slash amounts using:

  ```solidity
  vault.totalAssets()
  ```

  which **still includes pending withdrawals**.
* DSS can request slashing just after day 7, and finalize around day 9, slashing assets that belong to stakers whose withdrawals should be safe.
* If miner orders DSS's `finalizeSlashing()` before the user's `finishRedeem()`, the withdrawal gets unfairly reduced.

### Concrete Walkthrough

#### Scenario 1 - Vault over-slashed

* Vault = 100 tokens, one staker with 100 shares.
* Day 0: Staker withdraws 50 shares.
* Day 7: No DSS slashing request → withdrawal safe.
* Day 8: DSS requests slashing (50%).

  * Calculation: `50% * 100 = 50 tokens` (wrong).
  * Should be: `50% * (100 - 50 withdrawn) = 25 tokens`.
* Day 9: Staker redeems 50 tokens.
* Day 10: DSS finalizes → vault slashed 50 tokens (instead of 25).

#### Scenario 2 - Withdrawal unfairly slashed

* Vault = 150 tokens, A (100 shares) and B (50 shares).
* Day 0: A withdraws 100 shares.
* Day 7: No DSS request → withdrawal safe.
* Day 7 + 5 min: DSS requests 50% slash.

  * Slash calc: `50% * 150 = 75 tokens` (wrong).
  * Should be: `50% * (150 - 100) = 25 tokens`.
* Day 9: A redeems. DSS finalizes slashing first (75 tokens removed).
* A redeems only **50 tokens instead of 100**.

## Why This Matters

* **Violates withdrawal safety guarantee**: Users expect immunity after the 7-day slashing window, but in reality they're slashable until day 9.
* **Unfair loss of funds**: Stakers can lose part or all of their "safe" withdrawals.
* **Miner ordering risk**: Attackers can rely on block producers to prioritize slashing transactions.

## Vulnerable Code Reference

### 1) Slashing includes pending withdrawals

  ```solidity
  // SlasherLib.sol
  earmarkedStakes[i] = Math.mulDiv(
      slashingMetadata.slashPercentagesWad[i],
      IKarakBaseVault(slashingMetadata.vaults[i]).totalAssets(), // includes pending withdrawals
      Constants.MAX_SLASHING_PERCENT_WAD
  );
  ```

### 2) Withdrawal completion enforces 9-day delay even if safe

  ```solidity
  // VaultLib.sol
  if (qdWithdrawal.start + Constants.MIN_WITHDRAWAL_DELAY > block.timestamp) {
      revert MinWithdrawDelayNotPassed();
  }
  ```

### 3) Slashing finalization can front-run redeem

  ```solidity
  // Core.sol
  function finalizeSlashing(SlasherLib.QueuedSlashing memory queuedSlashing) external {
      _self().finalizeSlashing(queuedSlashing);
  }
  ```

## Recommended Mitigation

1. **Exclude pending withdrawals from slashable assets**:

   * Adjust `fetchEarmarkedStakes` to compute slashing over `totalAssets - pendingWithdrawals`.

2. **Early withdrawals if no slash request**:

   * Allow `finishRedeem()` after **7 days** if no slashing request was made during the slashing window.
   * Only enforce 9-day delay if slashing was requested.

3. **Transaction ordering defense**:

   * Consider checkpointing withdrawals as "immune" after 7 days so they cannot be slashed regardless of transaction ordering.

## Pattern Recognition Notes

* **Accounting Misalignment**: Using `totalAssets()` without subtracting locked/pending withdrawals introduces over-slashing risk.
* **Timing Window Exploits**: Safety guarantees (7-day slash window) must match accounting rules; otherwise veto periods introduce hidden risk.
* **Front-running / Miner Ordering**: Finalization transactions competing with redemptions can result in unfair priority outcomes.
* **Mitigation Principle**: Lock-up periods should be **aligned with slashable accounting**, or withdrawal safety rules become meaningless.

### Quick Recall (TL;DR)

* **Bug**: Pending withdrawals remain in `totalAssets()` → still slashable after the slashing window.
* **Impact**: DSS can slash more than allowed or reduce safe withdrawals if they finalize before redemption.
* **Fix**: Subtract pending withdrawals from slashing basis, and/or allow early finish if no slash request occurs within 7 days.
