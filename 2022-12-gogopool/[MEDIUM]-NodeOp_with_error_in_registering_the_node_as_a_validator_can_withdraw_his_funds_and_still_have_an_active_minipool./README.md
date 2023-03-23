# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/469
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L484
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L46


# Vulnerability details

## Impact

The ```MinipoolManager.sol::recordStakingError()``` function is called when there is an error while registering the node as a validator. The problem is that the ```recordStakingError()``` does not decrement the nodeop's minipool count as it is done in the [recordStakingEnd() function](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L437)

If the ```minipool``` count does not decrease, the [ClaimNodeOp.sol::isEligible()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L46) function could be unstable because the [staking.getRewardsStartTime()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L48) is not [set to zero here](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L83) because the nodeop's minipool is still more than zero.

The function ```ClaimNodeOp.sol::isEligible()``` can return a staker for eligible for rewards even if it is not the case.

## Proof of Concept

I created a test in ```ClaimNodeOp.t.sol```:

1. NodeOp1 creates minipool
2. Rialto calls claimAndInitiateStaking, recordStakingStart and recordStakingError
3. NodeOp1 withdraw his funds from minipool
4. NodeOp1 still has "active minipool" even when he withdraw his funds

```solidity
function testRecordStakingErrorAndWithdrawStillHaveActiveMinipool() public {
    // NodeOp with error in registering the node as a validator can withdraw his funds and
    // still have an active minipool
    // 1. NodeOp1 creates minipool
    // 2. Rialto/multisig claimAndInitiateStaking, recordStakingStart and recordStakingError
    // 3. NodeOp1 withdraw his funds from minipool
    // 4. NodeOp1 still has "active minipool"
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
    // 4. NodeOp1 still has "active minipool"
    //
    assertEq(staking.getMinipoolCount(nodeOp1), 1);
}
```

## Tools used

Vscode/Foundry

## Recommended Mitigation Steps

Add ```staking.decreaseMinipoolCount(owner);``` in the ```MinipoolManager.sol::recordStakingError()``` function.