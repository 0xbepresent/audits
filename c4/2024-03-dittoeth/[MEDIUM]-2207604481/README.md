# Original link
https://github.com/code-423n4/2024-03-dittoeth-findings/issues/33
# Lines of code

https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L259
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L151-L162


# Vulnerability details

## Impact

The report named `Valid redemption proposals can be disputed by decreasing collateral` explains how an attacker can steal from proposers by disputing valid redemption proposals. If any function that modifies a short records state (in a way that influences the CR) without updating the `updatedAt` param is executed. This report showcases another path that leads to such a state.

When bad debt occurs in the system it is socialized among all short records by increasing the `ercDebtRate`. At the next interaction with this short the `updateErcDebt` function is called to apply the portion of bad debt to the short:

```solidity
function updateErcDebt(STypes.ShortRecord storage short, address asset) internal {
    AppStorage storage s = appStorage();

    // Distribute ercDebt
    uint64 ercDebtRate = s.asset[asset].ercDebtRate;
    uint88 ercDebt = short.ercDebt.mulU88(ercDebtRate - short.ercDebtRate);

    if (ercDebt > 0) {
        short.ercDebt += ercDebt;
        short.ercDebtRate = ercDebtRate;
    }
}
```

As we can see the `updateErcDebt` function has the mentioned properties. It increases the debt and therefore influences the CR without updating the `updatedAt` param. This enables the following attack path:

- User creates a valid redemption proposal
- Bad debt occurs in the system
- The attacker applies the bad debt to a short record that is not part of the proposal and by doing so falls below the CR of any of the SRs in the proposal
- Attacker disputes the redemption proposal to receive the penalty fee

The `updateErcDebt` function is internal, but the `proposeRedemption` function can be used to apply it on any SR.

## Proof of Concept

The following POC can be implemented in the Redemption.t.sol test file and showcases that the `proposeRedemption` function can be used to apply bad debt to SRs without updating the updatedAt parameter. That this can be misused to steal funds as shown with another POC in the mentioned report.

```solidity
function test_proposeRedemption_does_update_updatedAt() public {
    // setup
    uint88 _redemptionAmounts = DEFAULT_AMOUNT * 2;
    makeShorts({singleShorter: true});

    skip(1 hours);
    _setETH(1000 ether);

    // propose a redemption
    MTypes.ProposalInput[] memory proposalInputs =
        makeProposalInputsForDispute({shortId1: C.SHORT_STARTING_ID, shortId2: C.SHORT_STARTING_ID + 1});

    uint32 updatedAtBefore = getShortRecord(sender, C.SHORT_STARTING_ID).updatedAt;

    vm.prank(receiver);
    diamond.proposeRedemption(asset, proposalInputs, _redemptionAmounts, MAX_REDEMPTION_FEE);

    uint32 updatedAtAfter = getShortRecord(sender, C.SHORT_STARTING_ID).updatedAt;

    // updatedAt param was not updated
    assertEq(updatedAtBefore, updatedAtAfter);
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Update the updatedAt parameter every time the `updateErcDebt` function is called, or/and call `updateErcDebt` in the `disputeRedemption` function on the SR at the `incorrectIndex` before comparing the CRs.



## Assessed type

Context