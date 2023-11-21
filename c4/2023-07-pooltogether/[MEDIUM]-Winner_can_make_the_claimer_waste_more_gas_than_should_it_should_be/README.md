# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/224
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L607
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1043


# Vulnerability details

## Impact

The winners have an option to [set up hooks](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L653) in order to execute their contracts [before](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/interfaces/IVaultHooks.sol#L22) and after the [claim prize](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/interfaces/IVaultHooks.sol#L34). When there is a claim, the [Vault._claimPrize()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1043) function search if [the winner has a configured hook](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1052-L1056), it calls the [winnerContractHook.beforeClaimPrize()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1053C40-L1053C56) functon then after the prize is claimed, it executes the [winnerContractHook.afterClaimPrize()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1068C28-L1068C43) function.

The problem is that all the transaction gas is sent to those external calls. Since there is not gas limit in each external call, the `Vault._claimPrize()` function may be reverted by out of gas error OR the claimer may waste more gas than it should be for the claim process.

Winners maliciously/accidentally can setup hooks that consume all the available gas or make claimer to waste more gas than it should be.

## Proof of Concept

The `1053 and 1068` code lines call the winner's contract without any limit of the gas sent to those external calls.

```solidity
File: Vault.sol
1043:   function _claimPrize(
1044:     address _winner,
1045:     uint8 _tier,
1046:     uint32 _prizeIndex,
1047:     uint96 _fee,
1048:     address _feeRecipient
1049:   ) internal returns (uint256) {
1050:     VaultHooks memory hooks = _hooks[_winner];
1051:     address recipient;
1052:     if (hooks.useBeforeClaimPrize) {
1053:       recipient = hooks.implementation.beforeClaimPrize(_winner, _tier, _prizeIndex);
1054:     } else {
1055:       recipient = _winner;
1056:     }
1057: 
1058:     uint prizeTotal = _prizePool.claimPrize(
1059:       _winner,
1060:       _tier,
1061:       _prizeIndex,
1062:       recipient,
1063:       _fee,
1064:       _feeRecipient
1065:     );
1066: 
1067:     if (hooks.useAfterClaimPrize) {
1068:       hooks.implementation.afterClaimPrize(
1069:         _winner,
1070:         _tier,
1071:         _prizeIndex,
1072:         prizeTotal - _fee,
1073:         recipient
1074:       );
1075:     }
1076: 
1077:     return prizeTotal;
1078:   }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Add gas limitation to each winner external call. Additionally, implements a function which helps the winner claim his own prize.


## Assessed type

call/delegatecall