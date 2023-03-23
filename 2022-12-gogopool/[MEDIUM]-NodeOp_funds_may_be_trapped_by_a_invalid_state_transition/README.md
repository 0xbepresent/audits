# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/723
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L484
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L528
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L287


# Vulnerability details

## Impact

The Multisig can call ```MinipoolManager.sol::recordStakingError()``` if there is an error while registering the node as a validator. Also the Multisig can call [MinipoolManager.sol::finishFailedMinipoolByMultisig()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L528) in order to "finish" the NodeOp's minipool proccess.

If the Multisig accidentally/intentionally calls ```recordStakingError()``` then ```finishFailedMinipoolByMultisig()``` the NodeOp funds may be trapped in the protocol.

The ```finishFailedMinipoolByMultisig()``` has the next comment: *Multisig can move a minipool from the error state to the finished state after a human review of the error* but the NodeOp should be able to withdraw his funds after a finished minipool.

## Proof of Concept

I created a test for this situation in ```MinipoolManager.t.sol```. At the end you can observe that the ```withdrawMinipoolFunds()``` reverts with ```InvalidStateTransition``` error:

1. NodeOp creates the minipool
2. Rialto calls claimAndInitiateStaking
3. Something goes wrong and Rialto calls recordStakingError()
4. Rialto accidentally/intentionally calls finishFailedMinipoolByMultisig() in order to finish the NodeOp's minipool
5. The NodeOp can not withdraw his funds. The withdraw function reverts with InvalidStateTransition() error

```solidity
function testUserFundsStuckErrorFinished() public {
    // NodeOp funds may be trapped by a invalid state transition
    // 1. NodeOp creates the minipool
    // 2. Rialto calls claimAndInitiateStaking
    // 3. Something goes wrong and Rialto calls recordStakingError()
    // 4. Rialto accidentally/intentionally calls finishFailedMinipoolByMultisig() in order 
    // to finish the NodeOp's minipool
    // 5. The NodeOp can not withdraw his funds. The withdraw function reverts with
    // InvalidStateTransition() error
    //
    // 1. Create the minipool by the NodeOp
    //
    address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
    vm.prank(liqStaker1);
    ggAVAX.depositAVAX{value: MAX_AMT}();
    assertEq(liqStaker1.balance, 0);
    uint256 duration = 2 weeks;
    uint256 depositAmt = 1000 ether;
    uint256 avaxAssignmentRequest = 1000 ether;
    uint256 validationAmt = depositAmt + avaxAssignmentRequest;
    uint128 ggpStakeAmt = 200 ether;
    vm.startPrank(nodeOp);
    ggp.approve(address(staking), ggpStakeAmt);
    staking.stakeGGP(ggpStakeAmt);
    MinipoolManager.Minipool memory mp = createMinipool(depositAmt, avaxAssignmentRequest, duration);
    vm.stopPrank();
    assertEq(vault.balanceOf("MinipoolManager"), depositAmt);
    //
    // 2. Rialto calls claimAndInitiateStaking
    //
    vm.startPrank(address(rialto));
    minipoolMgr.claimAndInitiateStaking(mp.nodeID);
    assertEq(vault.balanceOf("MinipoolManager"), 0);
    //
    // 3. Something goes wrong and Rialto calls recordStakingError()
    //
    bytes32 errorCode = "INVALID_NODEID";
    minipoolMgr.recordStakingError{value: validationAmt}(mp.nodeID, errorCode);
    // NodeOps funds should be back in vault
    assertEq(vault.balanceOf("MinipoolManager"), depositAmt);
    // 4. Rialto accidentally/intentionally calls finishFailedMinipoolByMultisig() in order 
    // to finish the NodeOp's minipool
    minipoolMgr.finishFailedMinipoolByMultisig(mp.nodeID);
    vm.stopPrank();
    // 5. The NodeOp can not withdraw his funds. The withdraw function reverts with
    // InvalidStateTransition() error
    vm.startPrank(nodeOp);
    vm.expectRevert(MinipoolManager.InvalidStateTransition.selector);
    minipoolMgr.withdrawMinipoolFunds(mp.nodeID);
    vm.stopPrank();
}
```

## Tools used

Foundry/Vscode

## Recommended Mitigation Steps

The ```withdrawMinipoolFunds``` could add another [requireValidStateTransition](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#LL290C3-L290C30) in order to allow the withdraw after the finished minipoool.