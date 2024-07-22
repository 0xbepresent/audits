# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/506
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L151-L154
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L168-L172


# Vulnerability details

To explain this issue, I will need to mention two things: the fee structure of the protocol and how closing a market works. Let's start with the fees.

Lenders earn interest with two different types of fees: Base interest and delinquency fee. The base interest depends on the annual interest rate of the market and it is paid by the borrower no matter what. On the other hand, the delinquency fee is a penalty fee and it is paid by the borrower if the reserves of the market drop below the required reserves amount.

The important part is how the penalty fees are calculated and I'll be focusing on penalty fees at the moment.  
Every market has a delinquency grace period, which is a period that is not penalized. If a market is delinquent but the grace period is not passed yet, there is no penalty fee. After the grace period is passed, the penalty fee is applied.

The most crucial part starts now: The penalty fee does not become 0 immediately after the delinquency is cured. The penalty fee is still being applied even after the delinquency is cured until the grace tracker counts down to zero.

An example from the protocol [gitbook/borrowers section is](https://wildcat-protocol.gitbook.io/wildcat/using-wildcat/day-to-day-usage/borrowers): *"Note: this means that if a markets grace period is 3 days, and it takes 5 days to cure delinquency, this means that* ***4*** *days of penalty APR are paid."*

Here you can find the code snippet of penalty fee calculation:  
[https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L89](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L89)

---

Now, let's check how to close a market and here is the `closeMarket()` function:

```solidity
file: WildcatMarket.sol
142.  function closeMarket() external onlyController nonReentrant {
143.    MarketState memory state = _getUpdatedState();
144.    state.annualInterestBips = 0;
145.    state.isClosed = true;
146.    state.reserveRatioBips = 0;
147.    if (_withdrawalData.unpaidBatches.length() > 0) {
148.      revert CloseMarketWithUnpaidWithdrawals();
149.    }
150.    uint256 currentlyHeld = totalAssets();
151.@>  uint256 totalDebts = state.totalDebts(); //@audit-issue Current debt is calculated with the current scaleFactor. It doesn't check if there are remaining "state.timeDelinquent" to pay penalty fees. 
152.    if (currentlyHeld < totalDebts) {
153.      // Transfer remaining debts from borrower
154.@>    asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld); //@audit remaining debt is transferred and market is closed, but if the market was delinquent for a while, debt will keep increasing. Total assets will not cover the total debt
155.    } else if (currentlyHeld > totalDebts) {
156.      // Transfer excess assets to borrower
157.      asset.safeTransfer(borrower, currentlyHeld - totalDebts);
158.    }
159.    _writeState(state);
160.    emit MarketClosed(block.timestamp);
161.  }
```

[https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142C1-L161C4](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142C1-L161C4)

While closing the market, the total debt is calculated and the required amount is transferred to the market. This way all debts are covered. However, the covered total debt is calculated with the current scale factor. As you can see above, this function does not check if there are still penalties to be paid. It should have checked the `state.timeDelinquent`.

If the `state.timeDelinquent > grace period` when closing the market *(which means the borrower still has more penalties to pay)*, the scale factor will keep increasing after every state update.

The borrower didn't pay the remaining penalties when closing the market, but who will pay it?

* Lenders will keep earning those penalty fees (*the base rate will be 0 after closing, but the penalty fee will still accumulate*)
    
* Lenders will start withdrawing their funds.
    
* All lenders except the last one will withdraw `the exact debt to the lender when closed + the penalty fee after closing`.
    
* The last lender will not even be able to withdraw `the exact debt to the lender when closed` because some portion of the funds dedicated to the last lender are already transferred to the previous lenders as the penalty fee.
    

The borrower might intentionally do it to escape from the penalty, or the borrower may not even be aware of the situation.

1. The borrower had a cash flow problem after taking the debt
    
2. The market stayed delinquent for a long time
    
3. The borrower found some funds
    
4. The borrower wanted to close the high-interest debts right after finding some funds
    
5. Immediately paid everything and closed the market while the market was still delinquent.
    
6. From the borrower's perspective, they paid all of their debt while closing the market.

7. But in reality, the borrower only paid the half of the penalty fee (while the counter was counting up). But the second half of the penalties, which will be accumulated while the counter was counting down, is not paid by the borrower. 
    
The protocol does not check if there are remaining penalties, and doesn't charge the borrower enough while closing the market.
 
I provided a coded PoC below that shows every step of the vulnerability.

## Impact
- The borrower will pay only half of the penalty while closing the market. 
- The other half of the penalty will keep accumulating.
- One of the lenders (the last one to withdraw) will have to cover those unpaid penalties.

Borrowers who are aware of this may create charming markets with lower base rate but higher penalty rate (They know they won't pay the half of it). 
Or the borrowers may not be aware of this, but the protocol doesn't take the required penalty from them. They "unintentionally" not pay the penalty, but the lender will have to cover it. 

## Proof of Concept
### Coded PoC

You can use the protocol's own test setup to prove this issue.  
\- Copy the snippet below, and paste it into the `WildcatMarket.t.sol` test file.  
\- Run it with `forge test --match-test test_closeMarket_withoutPaying_HalfofThePenalty -vvv`

```solidity
// @audit Not pay the half, leave it to the last lender
  function test_closeMarket_withoutPaying_HalfofThePenalty() external {
    // -----------------------------------------CHAPTER ONE - PREPARE--------------------------------------------------------------------------------
    // ------------------------------DEPOSIT - BORROW - WITHDRAW -> MARKET IS DELINQUENT-------------------------------------------------------------

    // Alice and Bob deposit 50k each, borrower borrows 80% 
    _authorizeLender(bob);
    vm.prank(alice);
    market.depositUpTo(50_000e18);
    vm.prank(bob);
    market.depositUpTo(50_000e18);
    vm.prank(borrower);
    market.borrow(80_000e18);
    
    // Alice and Bob request withdrawal for 10k each, reserve will be 0, market will be delinquent.
    vm.prank(alice);
    market.queueWithdrawal(10_000e18);
    vm.prank(bob);
    market.queueWithdrawal(10_000e18);
    // skip withdrawal batch duration
    skip(1 days);
    market.executeWithdrawal(alice, 86401); //86401 is the batch expiry. I hardoced it to make it shorter but it can also be found with _witdrawalData
    market.executeWithdrawal(bob, 86401);

    // Update the state. Market must be delinquent.
    market.updateState();
    MarketState memory state = market.previousState();
    assertTrue(state.isDelinquent);

    //----------------------------------------------CHAPTER TWO - ACTION------------------------------------------------------------------------------
    //----------------------------------CLOSE THE MARKET IMMEDIATELY AFTER PAYING DEBT----------------------------------------------------------------
    // Fast forward the time while delinquent to see the effect of delinquency penalty fees.
    skip(30 days);

    // Give some funds to the borrower to pay the debt while closing.
    asset.mint(address(borrower), 100_000e18);
    _approve(borrower, address(market), type(uint256).max);

    // We will close the market now. Save current state parameters just before closing.
    market.updateState();
    state = market.previousState();
    uint256 normalizedBalanceOfAliceBeforeClosing = state.normalizeAmount(market.scaledBalanceOf(alice));
    uint256 normalizedBalanceOfBobBeforeClosing = state.normalizeAmount(market.scaledBalanceOf(bob));

    uint256 totalDebtBeforeClosing = state.totalDebts();
    uint256 scaleFactorBeforeClosing = state.scaleFactor;
    console2.log("debt before closing: ", totalDebtBeforeClosing);
    console2.log("scale factor before closing: ", scaleFactorBeforeClosing);

    // Total debt before closing == normalized balance of Alice and Bob + unclaimed rewards + protocol fees.
    assertEq(totalDebtBeforeClosing, normalizedBalanceOfAliceBeforeClosing + normalizedBalanceOfBobBeforeClosing + state.normalizedUnclaimedWithdrawals + state.accruedProtocolFees);

    // Close the market.   
    vm.prank(address(controller));
    market.closeMarket();
    // Total asset in the market must be equal to the total debts. All debts are covered (ACCORDING TO CURRENT DEBT)
    assertEq(state.totalDebts(), market.totalAssets());

    //-----------------------------------------------CHAPTER THREE - SHOW IT-------------------------------------------------------------------------------
    //---------------------------------DEBT WILL KEEP ACCUMULATING BECAUSE OF THE REMANINING PENALTY FEES--------------------------------------------------
    // Fast forward 30 more days. 
    // Annual interest rate is updated to 0 when closing the market, but penalty fee keeps accumulating until the "state.timeDelinquent" goes toward 0.
    skip(30 days);

    // Update the state.
    market.updateState();
    state = market.previousState();
    uint256 totalDebtAfterClosing = state.totalDebts();
    uint256 scaleFactorAfterClosing = state.scaleFactor;

    // Debt and scale factor kept accumulating. Total debt is higher than the paid amount by borrower.
    assertGt(totalDebtAfterClosing, totalDebtBeforeClosing);
    assertGt(scaleFactorAfterClosing, scaleFactorBeforeClosing);
    console2.log("debt after closing: ", totalDebtAfterClosing);
    console2.log("scale factor after closing: ", scaleFactorAfterClosing);

    // Who will pay this difference in debt? --> The last lender to withdraw from the market will cover it.
    // Previous lenders except the last one will keep earning those penalty fees, but the last one will have to pay those funds.

    // Alice withdraws all of her balance.
    uint256 normalizedBalanceOfAliceAfterClosing = state.normalizeAmount(market.scaledBalanceOf(alice));
    vm.prank(alice);
    market.queueWithdrawal(normalizedBalanceOfAliceAfterClosing);
    // withdrawal batch duration
    skip(1 days);
    market.executeWithdrawal(alice, 5356801); // 5356801 is the emitted batch expiry. I hardoced it to make it shorter but it can also be found with _witdrawalData

    // After Alice's withdrawal, there won't be enough balance in the market to fully cover Bob.
    // Bob had to pay the penalty fee that the borrower didn't pay
    uint256 normalizedBalanceOfBobAfterClosing = state.normalizeAmount(market.scaledBalanceOf(bob));
    assertGt(normalizedBalanceOfBobAfterClosing, market.totalAssets());
    console2.log("total assets left: ", market.totalAssets());
    console2.log("normalized amount bob should get: ", normalizedBalanceOfBobAfterClosing);
  }
```

Below, you can find the test results:

```solidity
Running 1 test for test/market/WildcatMarket.t.sol:WildcatMarketTest
[PASS] test_closeMarket_withoutPaying_HalfofThePenalty() (gas: 714390)
Logs:
  debt before closing:  81427089816031080808713
  scale factor before closing:  1016988862478541592821945607
  debt after closing:  82095794821496423225911
  scale factor after closing:  1025347675046858373036920502
  total assets left:  40413182814156745887236
  normalized amount bob should get:  41013907001874334921477

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.41ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used
Manuel review, Foundry

## Recommended Mitigation Steps
I think there might be two different solutions: Easier one and the other one.

The easy solution is just not to allow the borrower to close the market until all the penalty fees are accumulated. This can easily be done by checking `state.timeDelinquent` in the `closeMarket()` function.

That one is simple, but I don't think it is fair for the borrower because the borrower will have to pay the base rate too for that additional amount of time. Maybe the borrower will be inclined to pay the `current debt + future penalties` and close the market as soon as possible.

That's why I think closing the market can still be allowed even if there are penalties to accumulate. However, the problem with that one is we can not know the exact amount of future penalties due to the compounding mechanism. It will depend on how many times the state is updated while the grace counter counts down.

Therefore I believe a buffer amount should be added. If the borrowers want to close the market, they should pay `current debt + expected future penalties + buffer amount`, and the excess amount from the buffer should be transferred back to the borrower after every lender withdraws their funds.





## Assessed type

Context