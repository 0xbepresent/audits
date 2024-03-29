# Original link
https://github.com/sherlock-audit/2023-02-openq-judging/issues/383
0xbepresent

high

# ```fundingTotals``` is not updated when funder withdraw his funding in the ```TieredPercentage``` bounty.

## Summary

The refund action can be at any time after the expiration time. While the payment for the winners are processing the funder can refund his funding and the [fundingTotals](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L116) is not updated.

## Vulnerability Detail

The refund action [DepositManagerV1.sol::refundDeposit()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152) can be at any time after the expiration time. If there is a payment proccess for the winners via [ClaimManagerV1.sol::claimBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L31) or [ClaimManagerV1.sol::permissionedClaimTieredBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L75) the refund action can destabilize the protocol. See the next situation:

Tiered percentage bounty with 3 winners:
1. Funder1 funds 100 tokens.
2. Funder2 funds 100 tokens.
3. The payment is processed for the first winner via [claimTiered](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L104).
4. Funder2 withdraw his deposit.
5. The [fundingTotals](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L116) variable which has the token balance is not updated.
4. The calculation for the second winner will use a wrong ```fundingTotals``` because is not the same token balance. 

## Impact

A funder intentionally/accidentally can withdraw his funding before the winner claim his rewards causing a destabilization in the ```fundingTotals``` variable, therefore the next winners can not claim their rewards.

I created a test in ```ClaimManager.test.js```.

Steps:

1. Create the Tiered Percentage Bounty
2. Fund the tiered bounty. Two deposit 100 token each one. 200 total.
3. Claim for the first place winner. The reward amount is 60 token because it is the 30% of 200 tokens.
4. Refund the first deposit.
5. Second place can not get his reward by an error.
6. The bounty token balance is 40 tokens.

```javascript
it('stop refunds in order to protect winners', async () => {
    // 1. Create the Tiered Percentage Bounty
    // 2. Fund the tiered bounty. Two deposit 100 token each one.
    // 3. Claim for the first place winner. The reward amount is 60 because it is the 30% of 200 tokens.
    // 4. Refund the first deposit.
    // 5. Second place can not get his reward by an error.
    // 6. The bounty token balance still has unclaimed 40 tokens.
    //
    // 1. Create the Tiered Percentage Bounty
    //
    await openQProxy.mintBounty(
        Constants.bountyId,
        Constants.organization,
        tieredPercentageBountyInitOperation_permissioned_withoutAnyRequiredValue);
    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
    const bounty = await TieredPercentageBountyV1.attach(bountyAddress);
    //
    // 2. Fund the tiered bounty. Two deposit 100 token each one.
    //
    await mockLink.approve(bountyAddress, 10000000);
    await depositManager.fundBountyToken(
        bountyAddress,
        mockLink.address,
        100,
        1,
        Constants.funderUuid);
    const funderUuid2 = 'mock-funder-uuid2';
    await depositManager.fundBountyToken(
        bountyAddress,
        mockLink.address,
        100,
        1,
        funderUuid2);
    //
    // 3. Claim for the first place winner.
    //
    let abiEncodedTieredCloserDataFirstPlace = abiCoder.encode(
        ['address', 'string', 'address', 'string', 'uint256'],
        [
            claimant.address,
            "FlacoJones",
            claimant.address,
            "https://github.com/OpenQDev/OpenQ-Frontend/pull/398",
            0
        ]);
    await claimManager.connect(oracle).claimBounty(
        bountyAddress,
        claimant.address,
        abiEncodedTieredCloserDataFirstPlace);
    expect((
        await mockLink.balanceOf(claimant.address)).toString()).to.equal('60');//30% of 200.
    //
    // 4. Refund the first deposit
    //
    const bountyDepositId = generateDepositId(Constants.bountyId, 0);
    await expect(
        depositManager.refundDeposit(bountyAddress, bountyDepositId));
    //
    // 5. Second place can not get his reward by an error.
    //
    let abiEncodedTieredCloserDataSecondPlace = abiCoder.encode(
        ['address', 'string', 'address', 'string', 'uint256'],
        [
            claimantSecondPlace.address,
            "FlacoJones",
            claimantSecondPlace.address,
            "https://github.com/OpenQDev/OpenQ-Frontend/pull/391",
            1
        ]);
    await expect(claimManager.connect(oracle).claimBounty(
        bountyAddress,
        claimantSecondPlace.address,
        abiEncodedTieredCloserDataSecondPlace)).to.be.revertedWith("ERC20: transfer amount exceeds balance");
    //
    // 6. The bounty token balance still has unclaimed 40 tokens.
    //
    expect((
        await mockLink.balanceOf(bountyAddress)).toString()).to.equal('40');
});
```

## Code Snippet

The ```claimTiered()``` function is using a ```fundingTotals``` variable which is not updated in the refund action.

```solidity
File: TieredPercentageBountyV1.sol
104:     function claimTiered(
105:         address _payoutAddress,
106:         uint256 _tier,
107:         address _tokenAddress
108:     ) external onlyClaimManager nonReentrant returns (uint256) {
109:         require(
110:             bountyType == OpenQDefinitions.TIERED_PERCENTAGE,
111:             Errors.NOT_A_TIERED_BOUNTY
112:         );
113:         require(!tierClaimed[_tier], Errors.TIER_ALREADY_CLAIMED);
114: 
115:         uint256 claimedBalance = (payoutSchedule[_tier] *
116:             fundingTotals[_tokenAddress]) / 100;
117: 
118:         _transferToken(_tokenAddress, claimedBalance, _payoutAddress);
119:         return claimedBalance;
120:     }
```


## Tool used

Vscode

## Recommendation

Ensure that once the first place winner is processed the funders refunds can not be possible. Or update the ```fundingTotals``` in the refund action.

Duplicate of #266 