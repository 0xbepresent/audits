# Original link
https://github.com/sherlock-audit/2023-03-teller-judging/issues/232
0xbepresent

medium

# If the loan is into default, an attacker can force to the lender to receive the collateral instead the settlement amount

## Summary

The [withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L250) function is too permissive because the function allows to anyone to call it if the loan is into default causing the lender to receive the collateral instead the settlement amount. The lender should have the decision to take the collateral or wait until someone liquidates the loan. 

## Vulnerability Detail

The [withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L250) function helps to withdraw the collateral once the loan is into default. The ```withdraw()``` function can be called by anyone or when someone liquidates the loan via [liquidateCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/TellerV2.sol#L701) function.

The problem is that the ```withdraw()``` function is too permissive. A malicious actor can call it before anyone can liquidate the loan causing the lender to receive the collateral instead the settlement amount.

## Impact

A malicious actor can call ```withdraw()``` before someone can liquidate the loan causing the lender to receive the collateral instead the liquidation amount. The lender should have the decision to take the collateral or wait until someone liquidates the loan. In this case, the lender should have the option to wait if someone liquidates the loan with a favorable amount or take the collateral.

I created a test where the malicious actor calls ```withdraw()``` function causing the lender to receive the collateral instead the liquidation amount. Test steps:

1. Create the collateral bid and lender accept it
2. Assert the escrow has the borrower's collateral 
3. Skip time so the loan is defaulted
4. The liquidation is blocked because the necessary time has not passed
5. The withdraw collateral is possible so the lender receives the collateral instead the liquidation amount.

```solidity
File: TellerV2_Test.sol
505:     function test_AttackerCanForceTheLenderToReceiveCollateralInsteadLiquidationAmount() public {
506:         // An attacker can force to the lender to receive the collateral
507:         // instead the liquidation amount.
508:         // 1. Create the collateral bid and lender accept it
509:         // 2. Assert the escrow has the borrower's collateral 
510:         // 3. Skip time so the loan is defaulted
511:         // 4. The liquidation is blocked because the necessary time has not passed
512:         // 5. The withdraw collateral is possible so the lender receives the collateral instead the liquidation
513:         // amount.
514:         //
515:         // 1. Create the collateral bid and lender accept it
516:         //
517:         // Submit bid as borrower
518:         uint256 bidId = submitCollateralBid();
519:         // Accept bid as lender
520:         acceptBid(bidId);
521:         //
522:         // 2. Assert the escrow has the borrower's collateral
523:         //
524:         // Get newly created escrow and test the creates escrow has the same bidId and collateral stored
525:         address escrowAddress = collateralManager._escrows(bidId);
526:         CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);
527:         uint256 storedBidId = escrow.getBid();
528:         assertEq(bidId, storedBidId, "Collateral escrow was not created");
529:         uint256 escrowBalance = wethMock.balanceOf(escrowAddress);
530:         assertEq(collateralAmount, escrowBalance, "Collateral was not stored");
531:         //
532:         // 3. Skip time so the loan is defaulted
533:         //
534:         uint32 lastRepaid = tellerV2.lastRepaidTimestamp(bidId);
535:         vm.warp(lastRepaid + 10000);
536:         //
537:         // 4. The liquidation is blocked because the necessary time has not passed
538:         //
539:         vm.expectRevert(stdError.arithmeticError);
540:         tellerV2.liquidateLoanFull(bidId);
541:         //
542:         // 5. The withdraw collateral is possible so the lender receives the collateral instead the liquidation
543:         // amount.
544:         //
545:         assertEq(tellerV2.isLoanDefaulted(bidId), true);
546:         collateralManager.withdraw(bidId);
547:     }
```

## Code Snippet

The [withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L250) function is too permissive allowing to anyone call it causing unauthorized decisions over the collateral.

```solidity
File: CollateralManager.sol
250:     function withdraw(uint256 _bidId) external {
251:         BidState bidState = tellerV2.getBidState(_bidId);
252:         if (bidState == BidState.PAID) {
253:             _withdraw(_bidId, tellerV2.getLoanBorrower(_bidId));
254:         } else if (tellerV2.isLoanDefaulted(_bidId)) {
255:             _withdraw(_bidId, tellerV2.getLoanLender(_bidId));
256:             emit CollateralClaimed(_bidId);
257:         } else {
258:             revert("collateral cannot be withdrawn");
259:         }
260:     }
```

## Tool used

Manual review

## Recommendation

The ```withdraw()``` function must be available only for the loan involved actors (borrower and lender).


Duplicate of #2
