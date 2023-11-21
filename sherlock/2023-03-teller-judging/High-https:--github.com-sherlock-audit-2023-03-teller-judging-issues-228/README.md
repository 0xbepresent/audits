# Original link
https://github.com/sherlock-audit/2023-03-teller-judging/issues/228
0xbepresent

high

# Malicious borrower can block liquidations causing the lender to receive neither the settlement amount nor the collateral

## Summary

 Once the loan has started the borrower can modify the [_bidCollaterals](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L27) using the [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) function which causes that if the loan goes into default, the loan can not be liquidated and the lender can not receive neither the settlement amount nor the collateral.

## Vulnerability Detail

Once the loan has started the [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) is available to be called by anyone, allowing to modify the ```_bidCollaterals```. Then, if the loan goes into default, the [_withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L393) will be reverted by "Withdraw amount cannot be zero" error.

Please see the next attack scenario:

1. The loan has started.
2. The loan is defaulted and the malicious borrower calls [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) and put the collateral amount to [zero](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L434).
3. The liquidator wants to liquidate via [liquidateLoanFull()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/TellerV2.sol#L676) but is not possible because the function is reverted by "Withdraw amount cannot be zero" error.
4. The lender wants to grab the collateral via [withdraw()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L250) functions but is still not possible because the funcstion is reverted by "Withdraw amount cannot be zero" error.

## Impact

The malicious borrower can block the liquidations and the lender collateral withdrawals causing the lender to receive neither the settlement amount nor the collateral. Additionally the borrower can gain time to repay his loan.

I created a test where the malicious borrower deny the liquidations and the collateral withdrawals. Test steps:

1. Create the collateral bid and the lender accept it
2. Assert the escrow has the borrower's collateral
3. Malicious borrower modifies to zero the collateral amount via the [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) function
4. Skip the time so the loan is defaulted
5. The liquidations are reverted by "Withdraw amount cannot be zero" error
6. The collateral withdrawals are reverted by "Withdraw amount cannot be zero" error

```solidity
File: TellerV2_Test.sol
451:     function test_MaliciousBorrowerCanBlockLiquidations() public {
452:         // Malicious borrower can block liquidations causing the lender
453:         // to receive neither the settlement amount nor the collateral
454:         // 1. Create the collateral bid and lender accept it
455:         // 2. Assert the escrow has the borrower's collateral
456:         // 3. Malicious borrower modifies to zero the collateral amount
457:         // 4. Skip the time so the loan is defaulted
458:         // 5. The liquidations are reverted by "Withdraw amount cannot be zero" error
459:         // 6. The collateral withdrawals are reverted by "Withdraw amount cannot be zero" error
460:         //
461:         //
462:         // 1. Create the collateral bid and lender accept it
463:         //
464:         // Submit bid as borrower
465:         uint256 bidId = submitCollateralBid();
466:         // Accept bid as lender
467:         acceptBid(bidId);
468:         //
469:         // 2. Assert the escrow has the borrower's collateral
470:         //
471:         // Get newly created escrow and test the creates escrow has the same bidId and collateral stored
472:         address escrowAddress = collateralManager._escrows(bidId);
473:         CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);
474:         uint256 storedBidId = escrow.getBid();
475:         assertEq(bidId, storedBidId, "Collateral escrow was not created");
476:         uint256 escrowBalance = wethMock.balanceOf(escrowAddress);
477:         assertEq(collateralAmount, escrowBalance, "Collateral was not stored");
478:         //
479:         // 3. Malicious borrower modifies to zero the collateral amount.
480:         //
481:         Collateral memory maliciousInfo;
482:         maliciousInfo._amount = 0; // <-- zero value
483:         maliciousInfo._tokenId = 0;
484:         maliciousInfo._collateralType = CollateralType.ERC20;
485:         maliciousInfo._collateralAddress = address(wethMock);
486:         collateralManager.commitCollateral(bidId, maliciousInfo);
487:         //
488:         // 4. Skip the time so the loan is defaulted
489:         //
490:         Payment memory amountOwed = tellerV2.calculateAmountOwed(bidId);
491:         vm.warp(30 days);
492:         //
493:         // 5. The liquidations are reverted by "Withdraw amount cannot be zero" error
494:         //
495:         IERC20(address(daiMock)).approve(address(tellerV2), amountOwed.principal + amountOwed.interest);
496:         vm.expectRevert(bytes("Withdraw amount cannot be zero"));
497:         tellerV2.liquidateLoanFull(bidId);
498:         //
499:         // 6. The collateral withdrawals are reverted by "Withdraw amount cannot be zero" error
500:         //
501:         vm.expectRevert(bytes("Withdraw amount cannot be zero"));
502:         collateralManager.withdraw(bidId);
503:     }
```

## Code Snippet

The [commitCollateral()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117) allows anyone to modify the collateral available.

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

In the attack scenario, the collateral is set to zero causing the liquidations or collateral withdrawals transactions will be reverted by [amount zero error](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/escrow/CollateralEscrowV1.sol#L89).

```solidity
File: CollateralEscrowV1.sol
84:     function withdraw(
85:         address _collateralAddress,
86:         uint256 _amount,
87:         address _recipient
88:     ) external virtual onlyOwner {
89:         require(_amount > 0, "Withdraw amount cannot be zero");
90:         Collateral storage collateral = collateralBalances[_collateralAddress];
```

## Tool used

Manual review

## Recommendation

Don't allow collateral modifications once the loan has started.

Duplicate of #168 
