# Original link
https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/361
# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L386-L387


# Vulnerability details

## Impact

The user can lost his rewards if the reward token is removed from the ```producerTokens[producerToken].rewardTokens``` list. If the reward token is removed, the rewardToken length is [going to be zero](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L387), the user rewards [going to be zero](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L391) and the [for statement will never enter](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L396) so the user will lost his rewards for this token and the [global state rewards](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L390) will save inaccurate information.

## Proof of Concept

1. Add user pxGMX token an accrue his rewards
2. Remove rewardToken
3. The user won't get his rewards even if the reward token is added again

I did a test for this situation:
```solidity
// test/PirexRewards.t.sol
function testClaimRewardsWithRemovedToken(
    uint32 secondsElapsed,
    uint8 multiplier,
    bool useETH
) external {
    vm.assume(secondsElapsed > 10);
    vm.assume(secondsElapsed < 365 days);
    vm.assume(multiplier != 0);
    vm.assume(multiplier < 10);
    address userX = testAccounts[0];
    uint256 gmxAmount = 100e18;

    _mintApproveGmx(gmxAmount, address(this), address(pirexGmx), gmxAmount);
    pirexGmx.depositGmx(gmxAmount, userX);

    assertEq(weth.balanceOf(userX), 0);

    vm.warp(block.timestamp + secondsElapsed);

    //
    // Add reward token and harvest rewards from Pirex contract
    //
    pirexRewards.addRewardToken(pxGmx, weth);
    pirexRewards.harvest();

    pirexRewards.userAccrue(pxGmx, userX);
    (, , uint256 globalRewardsBeforeClaimPxGmx) = _getGlobalState(
        pxGmx
    );
    (, , uint256 userRewardsBeforeClaimPxGmx) = pirexRewards
        .getUserState(pxGmx, userX);
    //
    // Sum of reward amounts that the user is entitled to
    //
    uint256 expectedClaimAmount = ((pirexRewards.getRewardState(
        pxGmx,
        weth
    ) * _calculateUserRewards(pxGmx, userX)) /
        _calculateGlobalRewards(pxGmx));

    //
    // Remove WETH from pxGMX
    //
    vm.expectEmit(true, false, false, true, address(pirexRewards));
    emit RemoveRewardToken(pxGmx, 0);
    pirexRewards.removeRewardToken(pxGmx, 0);

    // vm.expectEmit(true, false, false, false, address(pirexGmx));
    // emit ClaimUserReward(userX, address(0), 0, 0, 0);  // this emit is not called
    vm.expectEmit(true, true, false, true, address(pirexRewards));
    emit Claim(pxGmx, userX);
    pirexRewards.claim(pxGmx, userX);

    (, , uint256 globalRewardsAfterClaimPxGmx) = _getGlobalState(pxGmx);
    (, , uint256 userRewardsAfterClaimPxGmx) = pirexRewards.getUserState(pxGmx, userX);

    assertEq(
        globalRewardsBeforeClaimPxGmx - userRewardsBeforeClaimPxGmx,
        globalRewardsAfterClaimPxGmx
    );
    assertEq(0, userRewardsAfterClaimPxGmx);

    //
    // Owner weth balance has not changed. It is still zero.
    // userX has not received their rewards
    //
    assertEq(weth.balanceOf(userX), 0);
    assertTrue(expectedClaimAmount != weth.balanceOf(userX));

    // Add again the rewardToken
    pirexRewards.addRewardToken(pxGmx, weth);
    pirexRewards.harvest();
    
    // userX balance rewards is still zero
    pirexRewards.claim(pxGmx, userX);
    assertEq(weth.balanceOf(userX), 0);
}
```

## Tools used

Foundry/VSCode

## Recommended Mitigation Steps

Add a check if the reward token exists or not so the ```p.userStates[user].rewards``` is not zero if the reward token was removed:

```solidity
...
...
ERC20[] memory rewardTokens = p.rewardTokens;
uint256 rLen = rewardTokens.length;

// Claim should be skipped and not reverted on zero global/user reward
if (globalRewards != 0 && userRewards != 0 && rLen != 0) {
    // Update global and user reward states to reflect the claim
    p.globalState.rewards = globalRewards - userRewards;
    p.userStates[user].rewards = 0;
    ...
    ...
}
```