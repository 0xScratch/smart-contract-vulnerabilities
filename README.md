# Smart Contract Vulnerabilities

## Access Control

|   Protocol    |   Vulnerability   |   Severity    |   Source  |
|---------------|-------------------|---------------|-----------|
|   BakerFi     |   [Arbitrary `originalAmount` in Flash Loan Data Allows Logic Manipulation](/bugs/access-control/bakerfi-flashloandata-callback-unvalidated-param.md) |   Medium  |   Code4rena   |
|   Karak   |   [Unslashable NativeVault via Unvalidated `extraData` and Unrestricted Manager Upgrade](/bugs/access-control/karak-nativevault-unslashable-vault-manager-upgrade.md) |   High    |   Code4rena  |
|   Stader  |   [Loss of Admin via Self-Assignment in `updateAdmin` (Role Revocation Bug)](/bugs/access-control/stader-staderconfig-updateadmin-same-address-access-loss.md)    |   Medium  |   Code4rena   |
|   Stader  |   [Permissionless Reward Drain Allows Unfair Operator Slashing in ValidatorWithdrawalVault](/bugs/access-control/stader-validatorwithdrawalvault-distributerewards-slashing.md)   |   Medium  |   Code4rena   |
|   Virtuals Protocol   |   [ContributionNft Mint Abuse via Unrestricted Proposer Control](/bugs/access-control/virtuals-contributionnft-mint-role-overreach.md)    |   Medium  |   Code4rena   |
