# Original link
https://github.com/sherlock-audit/2023-06-tokemak-judging/issues/492
0xbepresent

medium

# The `AbstractRewarder::queueNewRewards()` will transfer from the caller the incorrect rewards amount causing the liquidation process may be stuck and the vaults' rewarder not to receive rewards
## Summary

The `AbstractRewarder::queueNewRewards()` will transfer a an incorrect amount of tokens from the caller causing that the `LiquidationRow::liquidateVaultsForToken()` process may be stuck.

## Vulnerability Detail

The [AbstractRewarder::queueNewRewards()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L235) function helps transfer rewards from the caller and queue the rewards to the stakers. If the accrued rewards are too large relative to the new rewards (i.e., queuedRatio is greater than newRewardRatio), the new rewards will be added to the queue rather than being immediately distributed ([code block 245-254](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L245-L254)). So the rewards are assigned in the `queuedRewards` variable in the [code line 253](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L253):

```solidity
File: AbstractRewarder.sol
235:     function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
...
...
244:         } else {
245:             uint256 elapsedBlock = block.number - (periodInBlockFinish - durationInBlock);
246:             uint256 currentAtNow = rewardRate * elapsedBlock;
247:             uint256 queuedRatio = currentAtNow * 1000 / newRewards;
248: 
249:             if (queuedRatio < newRewardRatio) {
250:                 notifyRewardAmount(newRewards);
251:                 queuedRewards = 0;
252:             } else {
253:                 queuedRewards = newRewards;
254:             }
255:         }
...
...
```

Additionally the function will transfer from the caller the `newRewards` variable amount:

```solidity
File: AbstractRewarder.sol
235:     function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
...
...
259:         // Transfer the new rewards from the caller to this contract.
260:         IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
...
...
```

The problem is that the function will transfer again the rewards assigned in the `queuedRewards` variable which is incorrect because the `queuedRewards` were transfer before. The function in the code line 236 will get the `queuedRewards`, then it will get the `newRewards` (code line 237), then in the 239 code line will sum the `newRewards + queuedRewards`. Then in the 260 code line the function will transfer from the caller the newRewards `(newRewards + queuedRewards)`. That is not correct because the queuedRewards were transfer before to the contract.

```solidity
File: AbstractRewarder.sol
235:     function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
236:         uint256 startingQueuedRewards = queuedRewards;
237:         uint256 startingNewRewards = newRewards;
238: 
239:         newRewards += startingQueuedRewards;
...
...
259:         // Transfer the new rewards from the caller to this contract.
260:         IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
...
...
```

I created the next test where the caller first deposits `50e18`, the function transfer from the caller to the contract `50e18`, then callers deposits again `30e18` and the function will transfer from the caller to the contract `30e18 + 50e18` which is incorrect because the first deposit amount `50e18` was transfered in the first deposit call so the second call must transfer only the `30e18` amount.

```solidity
// File: LMPVault-Withdraw.t.sol
// $ forge test --match-test "test_doubleRewardsQueueSpend" -vvv
// 
    function test_doubleRewardsQueueSpend() public {
        _asset.mint(address(this), 10000);
        _asset.approve(address(_lmpVault), 10000);
        // setup the LMPVault
        assertEq(_lmpVault.balanceOf(address(this)), 0);
        assertEq(_lmpVault.rewarder().balanceOf(address(this)), 0);
        _accessController.grantRole(Roles.DV_REWARD_MANAGER_ROLE, address(this));
        _lmpVault.rewarder().addToWhitelist(address(this));
        _toke.mint(address(this), 1000e18);
        _toke.approve(address(_lmpVault.rewarder()), 1000e18);
        //
        // Deposit 1 token just to initiatalize the rewardRate, currentRewards, lastUpdateBlock and periodInBlockFinish
        uint256 firstInitializationAmount = 1e18;
        _lmpVault.rewarder().queueNewRewards(firstInitializationAmount);
        //
        // Set newRewardRatio to zero so we can queue all the rewards. It just for the test purpose
        // of course the queuedRatio could be greater in normal situation making the rewards to be queued.
        address(_lmpVault.rewarder()).call(abi.encodeWithSignature("setNewRewardRate(uint256)", 0));
        //
        // Reward token is deposited to the rewarder contract (50e18) and 
        // The toke balance is decreased by 50e18.
        uint256 firstRewardDeposit = 50e18;
        _lmpVault.rewarder().queueNewRewards(firstRewardDeposit);
        assertEq(
            _toke.balanceOf(address(this)),
            1000e18 - firstInitializationAmount - firstRewardDeposit);
        //
        // Deposit more rewards (30e18). The problem is that the contract function will transfer from the
        // tokeBalance 50e18 (the first deposit amount) and 30e18 from this new deposit.
        uint256 secondRewardDeposit = 30e18;
        _lmpVault.rewarder().queueNewRewards(30e18);
        //
        // The toke balance is decreased by the first deposit amount 50e18
        // and another 50e18 and the 30e18 deposited tokens.
        // The contract function will transfer again the `firstRewardDeposit`.
        assertEq(
            _toke.balanceOf(address(this)),
            1000e18 - firstInitializationAmount - firstRewardDeposit - firstRewardDeposit - secondRewardDeposit);
    }
```

The [LiquidationRow::_performLiquidation()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L240) helps to collect rewards and [assign them to the vault's MainRewarder](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L267-L277) contract (AbstractRewarder.sol).

In the code line 276 the function approves an specific amount to the `AbstractRewarder.queueNewRewards()` function. So the liquidation process may be reverted because the `queueNewRewards()` function will transfer more tokens than the specified amount.

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

## Impact

The liquidation process may be stuck when vault's rewarder has queued rewards. This cause that rewards available in the `LiquidationRow` contract may not able to transfer vault's rewarder contract.

## Code Snippet

- The [AbstractRewarder.queueNewRewards()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L235) function.
- The [LiquidationRow._performLiquidation()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L240C14-L240C33) function.

## Tool used

Manual review

## Recommendation

The [AbstractRewarder.queueNewRewards()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L235) function must transfer from the caller only the new rewards not the queued rewards that were transfer before.

```diff
    function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
        uint256 startingQueuedRewards = queuedRewards;
        uint256 startingNewRewards = newRewards;
++      // Transfer the new rewards from the caller to this contract.
++      IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
        newRewards += startingQueuedRewards;

        if (block.number >= periodInBlockFinish) {
            notifyRewardAmount(newRewards);
            queuedRewards = 0;
        } else {
            uint256 elapsedBlock = block.number - (periodInBlockFinish - durationInBlock);
            uint256 currentAtNow = rewardRate * elapsedBlock;
            uint256 queuedRatio = currentAtNow * 1000 / newRewards;

            if (queuedRatio < newRewardRatio) {
                notifyRewardAmount(newRewards);
                queuedRewards = 0;
            } else {
                queuedRewards = newRewards;
            }
        }

        emit QueuedRewardsUpdated(startingQueuedRewards, startingNewRewards, queuedRewards);

--      // Transfer the new rewards from the caller to this contract.
--      IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
    }
```

Duplicate of #379
