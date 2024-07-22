# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/620
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L450


# Vulnerability details

## Impact
After recreation, minipools will receive more AVAX than the sum of their owners' current stake and the rewards that they generated.
## Proof of Concept
Multipools that successfully finished validation may be recreated by multisigs, before staked GGP and deposited AVAX have been withdrawn by minipool owners ([MinipoolManager.sol#L444](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L444)). The function compounds deposited AVAX by adding the rewards earned during previous validation periods to the AVAX amounts deposited and requested from stakers ([MinipoolManager.sol#L450-L452](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L450-L452)):
```solidity
Minipool memory mp = getMinipool(minipoolIndex);
// Compound the avax plus rewards
// NOTE Assumes a 1:1 nodeOp:liqStaker funds ratio
uint256 compoundedAvaxNodeOpAmt = mp.avaxNodeOpAmt + mp.avaxNodeOpRewardAmt;
setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpAmt")), compoundedAvaxNodeOpAmt);
setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerAmt")), compoundedAvaxNodeOpAmt);
```

The function assumes that a node operator and liquid stakers earned an equal reward amount: `compoundedAvaxNodeOpAmt` is calculated as the sum of the current AVAX deposit of the minipool owner and the node operator reward earned so far. However, liquid stakers get a smaller reward than node operators: the minipool node commission fee is applied to their share ([MinipoolManager.sol#L417](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L417)): 
```solidity
  uint256 avaxHalfRewards = avaxTotalRewardAmt / 2;

  // Node operators recv an additional commission fee
  ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
  uint256 avaxLiquidStakerRewardAmt = avaxHalfRewards - avaxHalfRewards.mulWadDown(dao.getMinipoolNodeCommissionFeePct());
  uint256 avaxNodeOpRewardAmt = avaxTotalRewardAmt - avaxLiquidStakerRewardAmt;
```

As a result, the `avaxLiquidStakerAmt` set in the `recreateMinipool` function will always be bigger than the actual amount since it equals to the compounded node operator amount, which includes node operator rewards.

Next, in the `recreateMinipool` function, the assigned AVAX amount is increased by the amount borrowed from liquid stakers + the node operator amount, which is again wrong because the assigned AVAX amount can only be increased by the liquid stakers' reward share ([MinipoolManager.sol#L457-L459](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L457-L459)):
```solidity
staking.increaseAVAXStake(mp.owner, mp.avaxNodeOpRewardAmt);
staking.increaseAVAXAssigned(mp.owner, compoundedAvaxNodeOpAmt);
staking.increaseMinipoolCount(mp.owner);
```

As a result, the amount of AVAX borrowed from liquid stakers by the minipool will be increased by the minipool node commission fee, the increased amount will be sent to the validator, and it will be required to end the validation period.

The following PoC demonstrates the wrong calculation:
```solidity
// test/unit/MinipoolManager.t.sol
function testRecreateMinipoolWrongLiquidStakerReward_AUDIT() public {
  uint256 duration = 4 weeks;
  uint256 depositAmt = 1000 ether;
  uint256 avaxAssignmentRequest = 1000 ether;
  uint256 validationAmt = depositAmt + avaxAssignmentRequest;
  // Enough to start but not to re-stake, we will add more later
  uint128 ggpStakeAmt = 100 ether;

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

  skip(duration / 2);

  // Give rialto the rewards it needs
  uint256 rewards = 10 ether;
  deal(address(rialto), address(rialto).balance + rewards);

  // Pay out the rewards
  vm.prank(address(rialto));
  minipoolMgr.recordStakingEnd{value: validationAmt + rewards}(mp.nodeID, block.timestamp, rewards);
  MinipoolManager.Minipool memory mpAfterEnd = minipoolMgr.getMinipoolByNodeID(mp.nodeID);
  assertEq(mpAfterEnd.avaxNodeOpAmt, depositAmt);
  assertEq(mpAfterEnd.avaxLiquidStakerAmt, avaxAssignmentRequest);

  // After the validation periods has ended, the node operator and liquid stakers got different rewards,
  // since a fee was taken from the liquid stakers' share.
  uint256 nodeOpReward = 5.75 ether;
  uint256 liquidStakerReward = 4.25 ether;
  assertEq(mpAfterEnd.avaxNodeOpRewardAmt, nodeOpReward);
  assertEq(mpAfterEnd.avaxLiquidStakerRewardAmt, liquidStakerReward);

  // Add a bit more collateral to cover the compounding rewards
  vm.prank(nodeOp);
  staking.stakeGGP(1 ether);

  vm.prank(address(rialto));
  minipoolMgr.recreateMinipool(mp.nodeID);

  MinipoolManager.Minipool memory mpCompounded = minipoolMgr.getMinipoolByNodeID(mp.nodeID);
  // After pool was recreated, node operator's amounts were increased correctly.
  assertEq(mpCompounded.avaxNodeOpAmt, mp.avaxNodeOpAmt + nodeOpReward);
  assertEq(mpCompounded.avaxNodeOpAmt, mp.avaxNodeOpInitialAmt + nodeOpReward);
  assertEq(staking.getAVAXStake(mp.owner), mp.avaxNodeOpAmt + nodeOpReward);

  // However, liquid stakers' amount were increased incorrectly: nodeOpReward was added instead of liquidStakerReward.
  // These assertions will fail:
  // expected: 1004.25 ether, actual 1005.75 ether
  assertEq(mpCompounded.avaxLiquidStakerAmt, mp.avaxLiquidStakerAmt + liquidStakerReward);
  assertEq(staking.getAVAXAssigned(mp.owner), mp.avaxLiquidStakerAmt + liquidStakerReward);
}
```

## Tools Used
Manual review
## Recommended Mitigation Steps
When compounding rewards in the `recreateMinipool` function, consider using an average reward so that node operator's and liquid stakers' deposits are increased equally and the entire reward amount is used:
```diff
--- a/contracts/contract/MinipoolManager.sol
+++ b/contracts/contract/MinipoolManager.sol
@@ -443,18 +443,22 @@ contract MinipoolManager is Base, ReentrancyGuard, IWithdrawer {
        /// @param nodeID 20-byte Avalanche node ID
        function recreateMinipool(address nodeID) external whenNotPaused {
                int256 minipoolIndex = onlyValidMultisig(nodeID);
                requireValidStateTransition(minipoolIndex, MinipoolStatus.Prelaunch);
                Minipool memory mp = getMinipool(minipoolIndex);
                // Compound the avax plus rewards
                // NOTE Assumes a 1:1 nodeOp:liqStaker funds ratio
-               uint256 compoundedAvaxNodeOpAmt = mp.avaxNodeOpAmt + mp.avaxNodeOpRewardAmt;
+               uint256 nodeOpRewardAmt = getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpRewardAmt")));
+               uint256 liquidStakerRewardAmt = getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerRewardAmt")));
+               uint256 avgRewardAmt = (nodeOpRewardAmt + liquidStakerRewardAmt) / 2;
+
+               uint256 compoundedAvaxNodeOpAmt = mp.avaxNodeOpAmt + avgRewardAmt;
                setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpAmt")), compoundedAvaxNodeOpAmt);
                setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerAmt")), compoundedAvaxNodeOpAmt);

                Staking staking = Staking(getContractAddress("Staking"));
                // Only increase AVAX stake by rewards amount we are compounding
                // since AVAX stake is only decreased by withdrawMinipool()
-               staking.increaseAVAXStake(mp.owner, mp.avaxNodeOpRewardAmt);
+               staking.increaseAVAXStake(mp.owner, avgRewardAmt);
                staking.increaseAVAXAssigned(mp.owner, compoundedAvaxNodeOpAmt);
                staking.increaseMinipoolCount(mp.owner);
```

Also, consider sending equal amounts of rewards to the vault and the ggAVAX token in the `recordStakingEnd` function.