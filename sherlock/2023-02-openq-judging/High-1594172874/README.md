# Original link
https://github.com/sherlock-audit/2023-02-openq-judging/issues/309
0xbepresent

high

# The first assigned winner can close the competition via ```ClaimManagerV1.sol::permissionedClaimTieredBounty()``` even when the other winners are not assigned yet.

## Summary

The first assigned winner can close the competition with the [ClaimManagerV1.sol::permissionedClaimTieredBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L75) function, then the deposits are closed and the other winners could not receive more deposits.

## Vulnerability Detail

The ```ClaimManagerV1.sol::permissionedClaimTieredBounty()``` helps to the winner to claim his bounty. The problem is that if the first winner was assigned via [setTierWinner()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredBountyCore.sol#L59), the winner can intentionally/accidentally close the competition and the deposits are now closed because the competition is closed.

That's could be a problem because imagine the next steps:
1. Funder1 funds 60 tokens to the bounty competition.
2. The bounty issuer sets the first winner via ```setTierWinner()```.
3. The first winner calls ```permissionedClaimTieredBounty()``` and the competition is closed now.
4. Funder2 wants to fund more tokens but is not possible anymore because the competition is closed.
5. The other winners can not claim a bounty because there are not more funds.

## Impact

The first assigned winner can accidentally/intentionally close the competition and consequently close the deposits for the bounty contest, then the other winners can not have available funds from the bounty contest.

I created the next test in ```ClaimManager.test.js```. Test steps:

1. Create the Tiered Percentage Bounty without any required documents
2. Fund the Tiered bounty.
3. Set the winner to the Claimant address
4. As a claimant call permissionedClaimTieredBounty()
5. The claimant winner now has 60% of tokens.
6. The competition now is closed and the deposits are not allowed anymore.

```javascript
const tieredBountyInitOperationBuilderWithoutAnyRequiredValue = (tokenAddress) => {
	const tieredAbiEncodedParams = abiCoder.encode(
		['uint256[]', 'bool', 'address', 'uint256', 'bool', 'bool', 'bool', 'string', 'string', 'string'],
		[
			[
				60,
				30,
				10
			],
			false,
			tokenAddress,
			Constants.volume,
			false,
			false,
			false,
			Constants.mockOpenQId,
			Constants.alternativeName,
			Constants.alternativeLogo]
	);
	const tieredPercentageBountyInitOperationComplete = [Constants.TIERED_PERCENTAGE_CONTRACT, tieredAbiEncodedParams];
	return tieredPercentageBountyInitOperationComplete;
};
tieredPercentageBountyInitOperation_permissioned_withoutAnyRequiredValue = tieredBountyInitOperationBuilderWithoutAnyRequiredValue(mockLink.address)

it('close competition as the first assigned winner', async () => {
    // As the first assigned Winner close the competition
    // 1. Create the Tiered Percentage Bounty without any required documents
    // 2. Fund the Tiered bounty.
    // 3. Set the winner to the Claimant address
    // 4. As a claimant call permissionedClaimTieredBounty()
    // 5. The claimant winner now has 60% of tokens.
    // 6. The competition now is closed and the deposits are not allowed anymore.
    //
    // 1. Create the Tiered Percentage Bounty
    //
    await openQProxy.mintBounty(
        Constants.bountyId,
        Constants.organization,
        tieredPercentageBountyInitOperation_permissioned_withoutAnyRequiredValue);
    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
    const bounty = await TieredPercentageBountyV1.attach(bountyAddress);
    // 2. Fund the tiered bounty.
    const volume = 60;
    await mockLink.approve(bountyAddress, 10000000);
    await depositManager.fundBountyToken(
        bountyAddress,
        mockLink.address,
        volume,
        1,
        Constants.funderUuid);
    //
    // 2. Set the winner to the Claimant address
    //
    await openQProxy.connect(oracle).associateExternalIdToAddress(
        Constants.mockOpenQId, claimant.address)
    await openQProxy.setTierWinner(
        Constants.bountyId, 0, Constants.mockOpenQId)
    //
    // 3. As a claimant call permissionedClaimTieredBounty() in order to the money
    //
    await claimManager.connect(claimant).permissionedClaimTieredBounty(
        bountyAddress,
        abiEncodedTieredCloserDataFirstPlace);
    //
    // 4. The competition now is closed.
    //
    const isClosed = await bounty.status();
    await expect(isClosed).to.equal(1);
    //
    // 5. The claimant winner now has 60% tokens.
    //
    expect((await mockLink.balanceOf(claimant.address)).toString()).to.equal('36');//60% for the first tier.
    //
    // 6. The competition now is closed and the deposits are not allowed anymore.
    //
    await expect(depositManager.fundBountyToken(
        bountyAddress,
        mockLink.address,
        10,
        1,
        Constants.funderUuid)).to.be.revertedWith("CONTRACT_ALREADY_CLOSED");
});
```

## Code Snippet

```solidity
File: ClaimManagerV1.sol
069:     /// @notice Used for claimants who have:
070:     /// @notice A) Completed KYC with KYC DAO for their tier
071:     /// @notice B) Uploaded invoicing information for their tier
072:     /// @notice C) Uploaded any necessary financial forms for their tier
073:     /// @param _bountyAddress The payout address of the bounty
074:     /// @param _closerData ABI Encoded data associated with this claim
075:     function permissionedClaimTieredBounty(
076:         address _bountyAddress,
077:         bytes calldata _closerData
078:     ) external onlyProxy {
079:         IBounty bounty = IBounty(payable(_bountyAddress));
080: 
081:         (, , , , uint256 _tier) = abi.decode(
082:             _closerData,
083:             (address, string, address, string, uint256)
084:         );
085: 
086:         string memory closer = IOpenQ(openQ).addressToExternalUserId(
087:             msg.sender
088:         );
089: 
090:         require(
091:             keccak256(abi.encodePacked(closer)) !=
092:                 keccak256(abi.encodePacked('')),
093:             Errors.NO_ASSOCIATED_ADDRESS
094:         );
095: 
096:         require(
097:             keccak256(abi.encode(closer)) ==
098:                 keccak256(abi.encode(bounty.tierWinners(_tier))),
099:             Errors.CLAIMANT_NOT_TIER_WINNER
100:         );
101: 
102:         if (bounty.bountyType() == OpenQDefinitions.TIERED_FIXED) {
103:             _claimTieredFixedBounty(bounty, msg.sender, _closerData);
104:         } else if (bounty.bountyType() == OpenQDefinitions.TIERED_PERCENTAGE) {
105:             _claimTieredPercentageBounty(bounty, msg.sender, _closerData);
106:         } else {
107:             revert(Errors.NOT_A_COMPETITION_CONTRACT);
108:         }
109: 
110:         emit ClaimSuccess(
111:             block.timestamp,
112:             bounty.bountyType(),
113:             _closerData,
114:             VERSION_1
115:         );
116:     }
```

## Tool used

Vscode

## Recommendation

Allow the ```permissionedClaimTieredBounty()``` execution when all the winners are assigned correctly. I think when the [setPayoutSchedule()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L141) and [setPayoutScheduleFixed()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L138) are executed correctly.

Duplicate of #266 