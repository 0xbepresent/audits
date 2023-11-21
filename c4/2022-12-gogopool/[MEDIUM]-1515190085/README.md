# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/345
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L385
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Vault.sol#L173
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L670


# Vulnerability details

## Impact

The ```MinipoolManager.sol::createMinipool()``` receives the ```duration``` param which helps to know how much time the validator will be enabled.

If the minipool is created with a zero value ```duration``` and the multisig wants to call ```recordStakingEnd()``` function with zero value for the ```avaxTotalRewardAmt``` param, the ```recorStakingEnd()``` will revert the transaction because the duration is not correct and the [Vault.sol::transferToken()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Vault.sol#L173) receives a zero value amount causing it to reverse. The ```MinipoolManager.sol::slash()``` function uses the ```duration``` value and that causes the revert.

The multisig can't return the liquid staker/user funds because the ```recordStakingEnd()``` will be reverting the transaction.

## Proof of Concept

I did a test:

1. Create a minipool with duration zero
2. Multisig claimInit the staking and record the staking start
3. Multisig call the recordStakingEnd with zero rewards and the transaction will revert by "InvalidAmount" error.

```solidity
function testRecordStakingEndWithSlashAndDurationZero() public {
    // recordStaking reverts when duration is zero and rewards amount is zero
    // 1. Create a minipool with duration zero
    // 2. Multisig claimInit the staking and record the staking start
    // 3. Multisig call the recordStakingEnd with zero rewards and the transaction will revert
    uint256 duration = 0;
    uint256 depositAmt = 1000 ether;
    uint256 avaxAssignmentRequest = 1000 ether;
    uint256 validationAmt = depositAmt + avaxAssignmentRequest;
    uint128 ggpStakeAmt = 200 ether;
    //
    // 1. Create a minipool with duration zero
    //
    vm.startPrank(nodeOp);
    ggp.approve(address(staking), MAX_AMT);
    staking.stakeGGP(ggpStakeAmt);//Node deposit ggpStakeAmt
    MinipoolManager.Minipool memory mp1 = createMinipool(depositAmt, avaxAssignmentRequest, duration);//create minipool with 0 duration
    vm.stopPrank();
    address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
    vm.prank(liqStaker1);//liquid staker deposit avax
    ggAVAX.depositAVAX{value: MAX_AMT}();
    //
    // 2. Multisig claimInit the staking and record the staking start
    //
    vm.prank(address(rialto));
    minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
    bytes32 txID = keccak256("txid");
    vm.prank(address(rialto));
    minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);
    console.log("Before recordStakingEnd AvaxLiquidStakerAmt:", mp1.avaxLiquidStakerAmt / 1 ether);
    console.log("Before recordStakingEnd getAVAXAssigned", staking.getAVAXAssigned(mp1.owner) / 1 ether);
    skip(2 weeks);
    //
    // 3. Multisig call the recordStakingEnd with zero rewards and the transaction will revert
    //
    vm.prank(address(rialto));
    vm.expectRevert(MinipoolManager.InvalidAmount.selector);
    minipoolMgr.recordStakingEnd{value: validationAmt}(mp1.nodeID, block.timestamp, 0 ether);
}
```

## Tools used

VsCode/Foundry

## Recommended Mitigation Steps

Validates a non-zero value duration in the ```createMinipool()``` function.