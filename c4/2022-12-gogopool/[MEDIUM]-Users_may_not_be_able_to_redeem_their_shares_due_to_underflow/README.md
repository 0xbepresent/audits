# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/317
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L191
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L88


# Vulnerability details

## Impact

The ```totalReleasedAssets``` variable is updated on the [syncRewards()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L88) function if someone calls the function before ```rewardsCycleEnd``` the [redeemAVAX()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L191) will be reverted because the ```totalReleasedAssets``` may not include all the rewards.

The ggAvax holder can not redeem his funds until the ```rewardsCycleEnd```

## Proof of Concept

I did the next test:

1. Create minipool (2000 avax)
2. Deposit rewards to the minipool (200 AVAX rewards)
3. Sync the rewards before the cycle ends
4. Redeem function will revert
5. Redeem will be available after the cycle end

```solidity
function testRedeemUnderOverFlow() public {
    // Redeem function reverts arithmetic error
    // 1.- Create minipool
    // 2.- Deposit rewards to the minipool
    // 3.- Sync the Rewards before the cycle end
    // 4.- Redeem function will revert
    // 5.- Redeem will be available after the cycle end.
    // Deposit liquid staker funds
    uint256 depositAmount = 1200 ether;
    uint256 nodeAmt = 2000 ether;
    uint128 ggpStakeAmt = 200 ether;
    vm.deal(bob, depositAmount);
    vm.prank(bob);
    ggAVAX.depositAVAX{value: depositAmount}();//Avax deposit 1200
    //
    // 1.- Create minipool
    //
    address nodeOp = getActorWithTokens("nodeOp", uint128(depositAmount), ggpStakeAmt);
    // Nodeop stake GGP and create minipoool
    vm.startPrank(nodeOp);
    ggp.approve(address(staking), ggpStakeAmt);
    staking.stakeGGP(ggpStakeAmt);
    MinipoolManager.Minipool memory mp = createMinipool(nodeAmt / 2, nodeAmt / 2, duration);
    vm.stopPrank();
    // Rialto init recordStakingStart
    vm.startPrank(address(rialto));
    minipoolMgr.claimAndInitiateStaking(mp.nodeID);
    minipoolMgr.recordStakingStart(mp.nodeID, randHash(), block.timestamp);
    vm.stopPrank();
    skip(mp.duration);
    //
    // 2.- Deposit rewards to the minipool
    //
    uint256 rewardsAmt = nodeAmt.mulDivDown(0.1 ether, 1 ether);
    console.log("Rewards amount:", rewardsAmt / 1 ether);
    vm.deal(address(rialto), address(rialto).balance + rewardsAmt);
    vm.prank(address(rialto));
    minipoolMgr.recordStakingEnd{value: nodeAmt + rewardsAmt}(mp.nodeID, block.timestamp, rewardsAmt);
    //
    // 3.- Sync the Rewards before the cycle end
    //
    ggAVAX.syncRewards();
    uint256 maxRedeemSharesBob = ggAVAX.maxRedeem(bob);
    console.log("TotalReleasedAssets after syncRewards:", ggAVAX.totalReleasedAssets() / 1 ether);
    console.log("LastRewards after syncRewards:", ggAVAX.lastRewardsAmt() / 1 ether);
    console.log("Bob maxRedeem():", maxRedeemSharesBob / 1 ether);
    //
    // 4.- Redeem function will revert
    //
    skip(1 days);
    console.log("Bob PreviewRedeem() after skip one day:", ggAVAX.previewRedeem(maxRedeemSharesBob) / 1 ether);
    vm.prank(bob);
    vm.expectRevert(stdError.arithmeticError); // Revert by arithmetic error
    ggAVAX.redeemAVAX(maxRedeemSharesBob);
    //
    // 5.- Redeem will be available after the cycle end.
    //
    skip(ggAVAX.rewardsCycleLength() + 1 days);
    ggAVAX.syncRewards();
    maxRedeemSharesBob = ggAVAX.maxRedeem(bob);
    console.log("");
    console.log("TotalReleasedAssets after syncRewards:", ggAVAX.totalReleasedAssets() / 1 ether);
    console.log("LastRewards after syncRewards:", ggAVAX.lastRewardsAmt() / 1 ether);
    console.log("Bob maxRedeem():", maxRedeemSharesBob / 1 ether);
    console.log("Bob PreviewRedeem() after skip to the cycle end:", ggAVAX.previewRedeem(maxRedeemSharesBob) / 1 ether);
    vm.prank(bob);
    ggAVAX.redeemAVAX(maxRedeemSharesBob);
}
```

Output:

```
[PASS] testRedeemUnderOverFlow() (gas: 1244356)
Logs:
  Rewards amount: 200
  TotalReleasedAssets after syncRewards: 1200
  LastRewards after syncRewards: 85
  Bob maxRedeem(): 1200
  Bob PreviewRedeem() after skip one day: 1206
  
  TotalReleasedAssets after syncRewards: 1285
  LastRewards after syncRewards: 0
  Bob maxRedeem(): 1200
  Bob PreviewRedeem() after skip to the cycle end: 1285
```

## Tools used
Foundry/VsCode

## Recommended Mitigation Steps
Consider redeem the max available amount for the shares owner instead of revert. The ```maxRedeem()``` function amount is not the same as the ```previewRedeem()``` amount.
