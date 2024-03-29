# Original link
https://github.com/sherlock-audit/2023-06-tokemak-judging/issues/499
0xbepresent

high

# Rewards will not be distributed to the vault's rewarder due a malfunction in `LiquidationRow::_performLiquidation()`
## Summary

The `LiquidationRow::_performLiquidation()` function will not distribute the rewards to the vault's rewarder because the function does not transfer the reward tokens to the `AsyncSwapper` contract causing the staker will not receive rewards.

## Vulnerability Detail

The [LiquidationRow::_performLiquidation()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L240C14-L240C33) function helps to [transfer the collected rewards to the vault's rewarder](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L277).

```solidity
File: LiquidationRow.sol
240:     function _performLiquidation(
241:         uint256 gasBefore,
242:         address fromToken,
243:         address asyncSwapper,
244:         IDestinationVault[] memory vaultsToLiquidate,
245:         SwapParams memory params,
246:         uint256 totalBalanceToLiquidate,
247:         uint256[] memory vaultsBalances
248:     ) private {
...
...
265:         for (uint256 i = 0; i < length; ++i) {
266:             IDestinationVault vaultAddress = vaultsToLiquidate[i];
267:             IMainRewarder mainRewarder = IMainRewarder(vaultAddress.rewarder());
268: 
269:             if (mainRewarder.rewardToken() != params.buyTokenAddress) {
270:                 revert InvalidRewardToken();
271:             }
272: 
273:             uint256 amount = amountReceived * vaultsBalances[i] / totalBalanceToLiquidate;
274: 
275:             // approve main rewarder to pull the tokens
276:             LibAdapter._approve(IERC20(params.buyTokenAddress), address(mainRewarder), amount);
277:             mainRewarder.queueNewRewards(amount);
...
...
```

At the beginning of the function, it swap the accumulated rewards tokens to the vault's rewards token (code line 251):

```solidity
File: LiquidationRow.sol
240:     function _performLiquidation(
241:         uint256 gasBefore,
242:         address fromToken,
243:         address asyncSwapper,
244:         IDestinationVault[] memory vaultsToLiquidate,
245:         SwapParams memory params,
246:         uint256 totalBalanceToLiquidate,
247:         uint256[] memory vaultsBalances
248:     ) private {
249:         uint256 length = vaultsToLiquidate.length;
250:         // the swapper checks that the amount received is greater or equal than the params.buyAmount
251:         uint256 amountReceived = IAsyncSwapper(asyncSwapper).swap(params);
```

The [AsyncSwapper.swap()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/BaseAsyncSwapper.sol#L28-L32) function will check if the desired amount to sell is in the contract (code line 30):

```solidity
File: BaseAsyncSwapper.sol
19:     function swap(SwapParams memory swapParams) public virtual nonReentrant returns (uint256 buyTokenAmountReceived) {
...
... 
25:         IERC20 sellToken = IERC20(swapParams.sellTokenAddress);
26:         IERC20 buyToken = IERC20(swapParams.buyTokenAddress);
27: 
28:         uint256 sellTokenBalance = sellToken.balanceOf(address(this));
29: 
30:         if (sellTokenBalance < swapParams.sellAmount) {
31:             revert InsufficientBalance(sellTokenBalance, swapParams.sellAmount);
32:         }
...
...
```

The problem is that the `LiquidationRow._performLiquidation()` does not transfer the amount to sell to the `AsyncSwapper` and the `AsynSwapper` does not make a `transferFrom` call.

I created a test where `LiquidationRow::claimsVaultRewards()` will be reverted by a InsufficientBalance in the `AsyncSwapper` contract:

```solidity
// File: LiquidationRow.t.sol
// $ forge test --match-test "test_TransferRewardsToMainRewarderWillFaillUsingTheBaseAsynSwapper" -vvv
//
import { IAsyncSwapper, SwapParams } from "src/interfaces/liquidation/IAsyncSwapper.sol";

    function test_TransferRewardsToMainRewarderWillFaillUsingTheBaseAsynSwapper() public {
        // The LiquidationRow::_performLiquidation() does not transfer the rewards to the AsyncSwapper causing a revert of
        // the transaction.
        // 
        // Setup the baseAsyncSwapper
        BaseAsyncSwapper baseAsyncSwapper = new BaseAsyncSwapper(vm.addr(100));
        liquidationRow.addToWhitelist(address(baseAsyncSwapper));
        SwapParams memory swapParams =
            SwapParams(address(rewardToken2), 200, address(targetToken), buyAmount, new bytes(0), new bytes(0));
        liquidationRow.setFeeAndReceiver(feeReceiver, feeBps);
        _mockComplexScenario(address(testVault));
        IDestinationVault[] memory vaults = _initArrayOfOneTestVault();
        //
        // Claims rewards
        liquidationRow.claimsVaultRewards(vaults);
        //
        // The liquidateVaultsForToken() will be reverted by InsufficientBalance
        vm.expectRevert(abi.encodeWithSelector(IAsyncSwapper.InsufficientBalance.selector, 0, 200));
        liquidationRow.liquidateVaultsForToken(address(rewardToken2), address(baseAsyncSwapper), vaults, swapParams);
    }
```

## Impact

The collected rewards will not be received by the vault's rewarders. The [collected rewards](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L298) by the LiquidationRow contract will be stuck in the contract.

## Code Snippet

- The [LiquidationRow::_performLiquidation()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L240C14-L240C33) function.
- The [BaseAsyncSwapper::swap()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/BaseAsyncSwapper.sol#L19) function.

## Tool used

Manual review

## Recommendation

Transfer the sell amount to the `AsyncSwapper` contract and make sure the buy token amount is returned to the `LiquidationRow` contract:

```diff
File: LiquidationRow.sol
     function _performLiquidation(
         uint256 gasBefore,
         address fromToken,
         address asyncSwapper,
         IDestinationVault[] memory vaultsToLiquidate,
         SwapParams memory params,
         uint256 totalBalanceToLiquidate,
         uint256[] memory vaultsBalances
     ) private {
         uint256 length = vaultsToLiquidate.length;
         // the swapper checks that the amount received is greater or equal than the params.buyAmount
++       IERC20(fromToken).safeApprove(address(asyncSwapper), params.sellAmount);
         uint256 amountReceived = IAsyncSwapper(asyncSwapper).swap(params);
++       if (IERC20(params.buyTokenAddress).balanceOf(address(this)) < params.buyAmount) revert();
...
```

```diff
File: BaseAsyncSwapper.sol
     function swap(SwapParams memory swapParams) public virtual nonReentrant returns (uint256 buyTokenAmountReceived) {
         if (swapParams.buyTokenAddress == address(0)) revert TokenAddressZero();
         if (swapParams.sellTokenAddress == address(0)) revert TokenAddressZero();
         if (swapParams.sellAmount == 0) revert InsufficientSellAmount();
         if (swapParams.buyAmount == 0) revert InsufficientBuyAmount();
 
         IERC20 sellToken = IERC20(swapParams.sellTokenAddress);
         IERC20 buyToken = IERC20(swapParams.buyTokenAddress);
++       IERC20(sellToken).safeTransferFrom(msg.sender, address(this), swapParams.sellAmount);
         uint256 sellTokenBalance = sellToken.balanceOf(address(this));
 
         if (sellTokenBalance < swapParams.sellAmount) {
             revert InsufficientBalance(sellTokenBalance, swapParams.sellAmount);
         }
...
```

Duplicate of #205
