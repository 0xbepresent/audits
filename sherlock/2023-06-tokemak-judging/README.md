
[High](High-https:--github.com-sherlock-audit-2023-06-tokemak-judging-issues-344/README.md) - 0xbepresent - An attacker can steal `native` token from the `LMPVaultRouterBase` contract due `LMPVaultRouterBase::deposit()` malfunction

[High](High-https:--github.com-sherlock-audit-2023-06-tokemak-judging-issues-499/README.md) - 0xbepresent - Rewards will not be distributed to the vault's rewarder due a malfunction in `LiquidationRow::_performLiquidation()`

[High](High-https:--github.com-sherlock-audit-2023-06-tokemak-judging-issues-492/README.md) - 0xbepresent - The `AbstractRewarder::queueNewRewards()` will transfer from the caller the incorrect rewards amount causing the liquidation process may be stuck and the vaults' rewarder not to receive rewards

[High](High-https:--github.com-sherlock-audit-2023-06-tokemak-judging-issues-462/README.md) - 0xbepresent - `Destination` vault rewards will be lost if the `swap` action in the `LMPVault::withdraw()` get more assets than the anticipated
