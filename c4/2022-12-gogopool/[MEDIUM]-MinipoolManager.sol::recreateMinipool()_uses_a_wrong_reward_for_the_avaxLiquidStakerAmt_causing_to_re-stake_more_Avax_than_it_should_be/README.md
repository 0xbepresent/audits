# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/347
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L452
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L458


# Vulnerability details

## Impact

The ```recreateMinipool()``` helps to re-stake the minipool with all the compounding rewards. The problem is that the [avaxLiquidStakerAmt](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L452) is not using the correct value, the correct value is ```avaxLiquidStakerAmt + avaxLiquidStakerRewardAmt``` for the ```avaxLiquidStakerAmt``` variable.

When the multisig calls ```recreateMinipool()```, the AvaxAssigned/avaxLiquidStakerAmt are wrong causing the protocol to re-stake wrong values for the AvaxAssigned variable then the validator/node receives more Avax.

## Proof of Concept

I did a test where you can see that the AvaxLiquidStakerAmt is not compounding the correct rewards:

1. Create Minipool, claimInitiateStakin and recordStakingStart
2. Multisig call the recordStakingEnd with 1e18 of rewards. The liquid staker get 425000000000000000 as a rewards
```
    AvaxLiquidStakerAmt: 1000000000000000000000
    avaxLiquidStakerRewardAmt: 425000000000000000
```
3. Restake the minipool and see how the AvaxAssigned does not reflect the rewards gained before.
```
    AvaxLiquidStakerAmt: 1000575000000000000000
    avaxLiquidStakerRewardAmt: 0
```

As you can see, the ```AvaxLiquidStakerAmt``` at the end should be the sum of (1000000000000000000000 + 425000000000000000) not 1000575000000000000000.

```solidity
function testRecreateMinipoolWrongLiquidStakerAmt() public {
    // Rewards liquidStaker wrong calculation
    // 1. Create Minipool, claimInitiateStakin and recordStakingStart
    // 2. Multisig call the recordStakingEnd with 1e18 of rewards
    // 3. Restake the minipool and see how the AvaxAssigned does not reflect the rewards gained before
    uint256 duration = 4 weeks;
    uint256 depositAmt = 1000 ether;
    uint256 avaxAssignmentRequest = 1000 ether;
    uint256 validationAmt = depositAmt + avaxAssignmentRequest;
    uint128 ggpStakeAmt = 100 ether;
    //
    // 1. Create Minipool, claimInitiateStakin and recordStakingStart
    //
    vm.startPrank(nodeOp);
    ggp.approve(address(staking), MAX_AMT);
    staking.stakeGGP(ggpStakeAmt);
    MinipoolManager.Minipool memory mp = createMinipool(depositAmt, avaxAssignmentRequest, duration);
    vm.stopPrank();
    address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
    vm.prank(liqStaker1);
    ggAVAX.depositAVAX{value: MAX_AMT}();
    vm.prank(address(rialto));
    minipoolMgr.claimAndInitiateStaking(mp.nodeID);
    bytes32 txID = keccak256("txid");
    vm.prank(address(rialto));
    minipoolMgr.recordStakingStart(mp.nodeID, txID, block.timestamp);
    skip(4 weeks / 2);
    // Give rialto the rewards it needs
    uint256 rewards = 1 ether;
    deal(address(rialto), address(rialto).balance + rewards);
    console.log("Before recordStakingEnd avaxNodeOpAmt:", mp.avaxNodeOpAmt);
    console.log("Before recordStakingEnd AvaxLiquidStakerAmt:", mp.avaxLiquidStakerAmt);//1000 ether
    console.log("Before recordStakingEnd avaxNodeOpRewardAmt:", mp.avaxNodeOpRewardAmt);
    console.log("Before recordStakingEnd avaxLiquidStakerRewardAmt:", mp.avaxLiquidStakerRewardAmt);
    console.log("Before recordStakingEnd getAVAXAssigned", staking.getAVAXAssigned(mp.owner));
    // Pay out the rewards
    console.log("");
    console.log("Rewards 1e18:", rewards);
    vm.prank(address(rialto));
    minipoolMgr.recordStakingEnd{value: validationAmt + rewards}(mp.nodeID, block.timestamp, rewards);
    MinipoolManager.Minipool memory mp2 = minipoolMgr.getMinipoolByNodeID(mp.nodeID);
    console.log("Before recreateMinipool avaxNodeOpAmt:", mp2.avaxNodeOpAmt);
    console.log("Before recreateMinipool AvaxLiquidStakerAmt:", mp2.avaxLiquidStakerAmt);//1000 ether
    console.log("Before recreateMinipool avaxNodeOpRewardAmt:", mp2.avaxNodeOpRewardAmt);
    console.log("Before recreateMinipool avaxLiquidStakerRewardAmt:", mp2.avaxLiquidStakerRewardAmt);
    console.log("Before recreateMinipool getAVAXAssigned", staking.getAVAXAssigned(mp2.owner));
    console.log("");
    //
    // 3. Restake the minipool and see how the AvaxAssigned does not reflect the rewards gained before
    //
    // Add a bit more collateral to cover the compounding rewards
    vm.prank(nodeOp);
    staking.stakeGGP(1 ether);
    vm.prank(address(rialto));
    minipoolMgr.recreateMinipool(mp.nodeID);
    MinipoolManager.Minipool memory mpCompounded = minipoolMgr.getMinipoolByNodeID(mp.nodeID);
    console.log("After recreteMinipool avaxNodeOpAmt:", mpCompounded.avaxNodeOpAmt);
    console.log("After recreteMinipool AvaxLiquidStakerAmt:", mpCompounded.avaxLiquidStakerAmt);//1000 ether
    console.log("After recreateMinipool avaxNodeOpRewardAmt:", mpCompounded.avaxNodeOpRewardAmt);
    console.log("After recreateMinipool avaxLiquidStakerRewardAmt:", mpCompounded.avaxLiquidStakerRewardAmt);
    console.log("After recreateMinipool getAVAXAssigned", staking.getAVAXAssigned(mp.owner));
}
```

Output:

```bash
Running 1 test for test/unit/MinipoolManager.t.sol:MinipoolManagerTest
[PASS] testRecreateMinipoolWrongLiquidStakerAmt() (gas: 1468903)
Logs:
  Before recordStakingEnd avaxNodeOpAmt: 1000000000000000000000
  Before recordStakingEnd AvaxLiquidStakerAmt: 1000000000000000000000
  Before recordStakingEnd avaxNodeOpRewardAmt: 0
  Before recordStakingEnd avaxLiquidStakerRewardAmt: 0
  Before recordStakingEnd getAVAXAssigned 1000000000000000000000
  
  Rewards 1e18: 1000000000000000000
  Before recreateMinipool avaxNodeOpAmt: 1000000000000000000000
  Before recreateMinipool AvaxLiquidStakerAmt: 1000000000000000000000
  Before recreateMinipool avaxNodeOpRewardAmt: 575000000000000000
  Before recreateMinipool avaxLiquidStakerRewardAmt: 425000000000000000
  Before recreateMinipool getAVAXAssigned 0
  
  After recreteMinipool avaxNodeOpAmt: 1000575000000000000000
  After recreteMinipool AvaxLiquidStakerAmt: 1000575000000000000000
  After recreateMinipool avaxNodeOpRewardAmt: 0
  After recreateMinipool avaxLiquidStakerRewardAmt: 0
  After recreateMinipool getAVAXAssigned 1000575000000000000000
```

## Tools used

VsCode/Foundry

## Recommended Mitigation Steps

The [recreateMinipool](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L444) should calculate the ```avaxLiquidStakerAmt``` using the ```avaxLiquidStakerRewardAmt``` amount:

```solidity
diff --git a/contracts/contract/MinipoolManager.sol b/contracts/contract/MinipoolManager.sol
index 8563580..7b6afbd 100644
--- a/contracts/contract/MinipoolManager.sol
+++ b/contracts/contract/MinipoolManager.sol
@@ -448,14 +448,15 @@ contract MinipoolManager is Base, ReentrancyGuard, IWithdrawer {
                // Compound the avax plus rewards
                // NOTE Assumes a 1:1 nodeOp:liqStaker funds ratio
                uint256 compoundedAvaxNodeOpAmt = mp.avaxNodeOpAmt + mp.avaxNodeOpRewardAmt;
+               uint256 compoundedavaxLiquidStakerAmt = mp.avaxLiquidStakerAmt + mp.avaxLiquidStakerRewardAmt;
                setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpAmt")), compoundedAvaxNodeOpAmt);
-               setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerAmt")), compoundedAvaxNodeOpAmt);
+               setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerAmt")), compoundedavaxLiquidStakerAmt);
 
                Staking staking = Staking(getContractAddress("Staking"));
                // Only increase AVAX stake by rewards amount we are compounding
                // since AVAX stake is only decreased by withdrawMinipool()
                staking.increaseAVAXStake(mp.owner, mp.avaxNodeOpRewardAmt);
-               staking.increaseAVAXAssigned(mp.owner, compoundedAvaxNodeOpAmt);
+               staking.increaseAVAXAssigned(mp.owner, compoundedavaxLiquidStakerAmt);
                staking.increaseMinipoolCount(mp.owner);
 
                if (staking.getRewardsStartTime(mp.owner) == 0) {
```