# Original link
https://github.com/sherlock-audit/2023-06-tokemak-judging/issues/462
0xbepresent

medium

# `Destination` vault rewards will be lost if the `swap` action in the `LMPVault::withdraw()` get more assets than the anticipated
## Summary

The `LMPVault:withdraw()` function accumulates the `destination rewards` when shares are burned. Additionally, the `LMPVault:withdraw()` performs a `swap` action, then it is possible to get back more assets than the anticipated from the swap. If there are more assets from the swap, the accumulated rewards from the `destinationVault` will be lost.

## Vulnerability Detail

The [LMPVault::_withdraw()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L448) function helps to the user to withdraw his deposited assets. If the vault does not have enough tokens, then [it will burn some shares](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L470-L483) from the `destinationVault`.

When the shares are burned from the `destinationVault`, the rewards are acumulated in the `info.idleIncrease` variable (See the [487](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L485-L487) code line).

```solidity
File: LMPVault.sol
448:     function _withdraw(
449:         uint256 assets,
450:         uint256 shares,
451:         address receiver,
452:         address owner
453:     ) internal virtual returns (uint256) {
...
...
482:                 uint256 assetPreBal = _baseAsset.balanceOf(address(this));
483:                 uint256 assetPulled = destVault.withdrawBaseAsset(sharesToBurn, address(this));
484: 
485:                 // Destination Vault rewards will be transferred to us as part of burning out shares
486:                 // Back into what that amount is and make sure it gets into idle
487:                 info.idleIncrease += _baseAsset.balanceOf(address(this)) - assetPreBal - assetPulled;
488:                 info.totalAssetsPulled += assetPulled;
489:                 info.debtDecrease += totalDebtBurn;
...
...
```

Additionally, the [destVault.withdrawBaseAsset()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/DestinationVault.sol#L244) will [swap tokens in order to get the baseAsset](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/DestinationVault.sol#L273). It is possible to get back more assets than the anticipated from the swap action, the function will throw it to the `idleIncrease` variable (see the [494](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L494) code line).

```solidity
File: LMPVault.sol
448:     function _withdraw(
449:         uint256 assets,
450:         uint256 shares,
451:         address receiver,
452:         address owner
453:     ) internal virtual returns (uint256) {
...
...
491:                 // It's possible we'll get back more assets than we anticipate from a swap
492:                 // so if we do, throw it in idle and stop processing. You don't get more than we've calculated
493:                 if (info.totalAssetsPulled > info.totalAssetsToPull) {
494:                     info.idleIncrease = info.totalAssetsPulled - info.totalAssetsToPull;
495:                     info.totalAssetsPulled = info.totalAssetsToPull;
496:                     break;
497:                 }
...
...
```

The problem is that the `info.idleIncrease` is overwritten. So the rewards accumulated in the [487 code line](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L487) will be lost if the swap action get back more assets in the [494](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L494) code line.

```solidity
File: LMPVault.sol
448:     function _withdraw(
449:         uint256 assets,
450:         uint256 shares,
451:         address receiver,
452:         address owner
453:     ) internal virtual returns (uint256) {
...
...
482:                 uint256 assetPreBal = _baseAsset.balanceOf(address(this));
483:                 uint256 assetPulled = destVault.withdrawBaseAsset(sharesToBurn, address(this));
484: 
485:                 // Destination Vault rewards will be transferred to us as part of burning out shares
486:                 // Back into what that amount is and make sure it gets into idle
487:                 info.idleIncrease += _baseAsset.balanceOf(address(this)) - assetPreBal - assetPulled;
488:                 info.totalAssetsPulled += assetPulled;
489:                 info.debtDecrease += totalDebtBurn;
490: 
491:                 // It's possible we'll get back more assets than we anticipate from a swap
492:                 // so if we do, throw it in idle and stop processing. You don't get more than we've calculated
493:                 if (info.totalAssetsPulled > info.totalAssetsToPull) {
494:                     info.idleIncrease = info.totalAssetsPulled - info.totalAssetsToPull;
495:                     info.totalAssetsPulled = info.totalAssetsToPull;
496:                     break;
497:                 }
...
...
```

## Impact

Destination vault rewards will be lost.

## Code Snippet

- The [LMPVault::_withdraw()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVault.sol#L448) function.
- The [DestinationVault.withdrawBaseAsset()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/DestinationVault.sol#L244) function.

## Tool used

Manual review

## Recommendation

The `info.idleIncrease` should not be overwritten when the more assets are back from the swap. It should count the accumulated rewards:

```diff
File: LMPVault.sol
448:     function _withdraw(
449:         uint256 assets,
450:         uint256 shares,
451:         address receiver,
452:         address owner
453:     ) internal virtual returns (uint256) {
...
...
482:                 uint256 assetPreBal = _baseAsset.balanceOf(address(this));
483:                 uint256 assetPulled = destVault.withdrawBaseAsset(sharesToBurn, address(this));
484: 
485:                 // Destination Vault rewards will be transferred to us as part of burning out shares
486:                 // Back into what that amount is and make sure it gets into idle
487:                 info.idleIncrease += _baseAsset.balanceOf(address(this)) - assetPreBal - assetPulled;
488:                 info.totalAssetsPulled += assetPulled;
489:                 info.debtDecrease += totalDebtBurn;
490: 
491:                 // It's possible we'll get back more assets than we anticipate from a swap
492:                 // so if we do, throw it in idle and stop processing. You don't get more than we've calculated
493:                 if (info.totalAssetsPulled > info.totalAssetsToPull) {
-- 494:                  info.idleIncrease = info.totalAssetsPulled - info.totalAssetsToPull;
++ 494:                  info.idleIncrease = (info.totalAssetsPulled - info.totalAssetsToPull) + info.idleIncrease;
495:                     info.totalAssetsPulled = info.totalAssetsToPull;
496:                     break;
497:                 }
...
...
```

Duplicate of #5
