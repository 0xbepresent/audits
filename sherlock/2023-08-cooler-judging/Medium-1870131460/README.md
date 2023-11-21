# Original link
https://github.com/sherlock-audit/2023-08-cooler-judging/issues/102
0xbepresent

high

# The `rollLoan()` function must be called only by the borrower
## Summary

The lender can create new terms via `provideNewTermsForRoll()` and the same lender accept them using the `rollLoan()` function making the borrower to increase his loan interests.

## Vulnerability Detail

The lender can create new terms via [provideNewTermsForRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282C14-L282C36). Then those new terms are accepted using the [rollLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192C14-L192C22) function.

The problem is that the malicious lender can create new malicious terms and accept them without restriction making the borrower to pay more interests. See the next scenario:
1. The lender provide new terms for the loan using the `provideNewTermsForRoll()`. He increases the interests to 20%.
2. Now, the same lender calls the `rollLoan()` function and those [new terms are accepted](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L203).
3. Now the borrower has non-agreed terms.

## Impact

The malicious lender can increment the interests, then accept them via `rollLoan()` making the borrower to pay more interests. The borrower may lose his collateral due an unpayable debt.

## Code Snippet

The [provideNewTermsForRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282C14-L282C36) function:

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

The [rollLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192C14-L192C22) function:

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

## Tool used

Manual review

## Recommendation

The borrower should be the only who can accept the new lender's terms:

```diff
    function rollLoan(uint256 loanID_) external {
++      if (msg.sender != owner()) rever();
...
...
...
```

Duplicate of #26
