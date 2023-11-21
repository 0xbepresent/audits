# Original link
https://github.com/sherlock-audit/2023-08-cooler-judging/issues/99
0xbepresent

medium

# Malicious lender can make the borrower to pay non-agreed interests via frontrunning attack
## Summary

Malicious lender can make the borrower to pay non-agreed interests via frontrunning the `Cooler::rollLoan()` function.

## Vulnerability Detail

The lender can provide new terms for the loan via the [Cooler::provideNewTermsForRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282C14-L282C36) function. Then the borrower accepts the new terms using the [Cooler::rollLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192C14-L192C22) function.

Malicious lender can front run the borrower `rollLoan()` call and change the interests making the borrower to pay non-agreeed interests. Please see the next scenario:

1. Lender creates new terms for the loan using the [Cooler::provideNewTermsForRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282C14-L282C36) function. 
2. Borrower calls the [Cooler::rollLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192) function in order to accept new terms.
3. Malicious lender see the transaction and frontran it changing the terms (interests and duration) using the `provideNewTermsForRoll()` function.
4. The borrower calls finally is executed but using non-agreed terms.

## Impact

The borrower can lost the collateral because lender can increase the interests to a no-payable debt making the borrower to be unable to repay the loan.

## Code Snippet

The [Cooler::rollLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192C14-L192C22) function:

```solidity
File: Cooler.sol
192:     function rollLoan(uint256 loanID_) external {
193:         Loan memory loan = loans[loanID_];
194: 
195:         if (block.timestamp > loan.expiry) revert Default();
196:         if (!loan.request.active) revert NotRollable();
197: 
198:         // Check whether rolling the loan requires pledging more collateral or not (if there was a previous repayment).
199:         uint256 newCollateral = newCollateralFor(loanID_);
200:         uint256 newDebt = interestFor(loan.amount, loan.request.interest, loan.request.duration);
201: 
202:         // Update memory accordingly.
203:         loan.amount += newDebt;
204:         loan.collateral += newCollateral;
205:         loan.expiry += loan.request.duration;
206:         loan.request.active = false;
207: 
208:         // Save updated loan info in storage.
209:         loans[loanID_] = loan;
210: 
211:         if (newCollateral > 0) {
212:             collateral().safeTransferFrom(msg.sender, address(this), newCollateral);
213:         }
214: 
215:         // If necessary, trigger lender callback.
216:         if (loan.callback) CoolerCallback(loan.lender).onRoll(loanID_, newDebt, newCollateral);
217:     }
```

The [Cooler::provideNewTermsForRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282C14-L282C36) function:

```solidity
File: Cooler.sol
282:     function provideNewTermsForRoll(
283:         uint256 loanID_,
284:         uint256 interest_,
285:         uint256 loanToCollateral_,
286:         uint256 duration_
287:     ) external {
288:         Loan storage loan = loans[loanID_];
289: 
290:         if (msg.sender != loan.lender) revert OnlyApproved();
291: 
292:         loan.request =
293:             Request(
294:                 loan.amount,
295:                 interest_,
296:                 loanToCollateral_,
297:                 duration_,
298:                 true
299:             );
300:     }
```

## Tool used

Manual review

## Recommendation

Implement that the borrower can specify how much is willing to pay for interests and the duration.

```diff
--  function rollLoan(uint256 loanID_) external {
++  function rollLoan(uint256 loanID_, uint256 interest, uint256 duration) external {
        Loan memory loan = loans[loanID_];

        if (block.timestamp > loan.expiry) revert Default();
        if (!loan.request.active) revert NotRollable();
++      if (interest != loan.request.interest || duration != loan.request.duration) revert();

        // Check whether rolling the loan requires pledging more collateral or not (if there was a previous repayment).
        uint256 newCollateral = newCollateralFor(loanID_);
        uint256 newDebt = interestFor(loan.amount, loan.request.interest, loan.request.duration);

        // Update memory accordingly.
        loan.amount += newDebt;
        loan.collateral += newCollateral;
        loan.expiry += loan.request.duration;
        loan.request.active = false;

        // Save updated loan info in storage.
        loans[loanID_] = loan;

        if (newCollateral > 0) {
            collateral().safeTransferFrom(msg.sender, address(this), newCollateral);
        }

        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onRoll(loanID_, newDebt, newCollateral);
    }
```



Duplicate of #243 
