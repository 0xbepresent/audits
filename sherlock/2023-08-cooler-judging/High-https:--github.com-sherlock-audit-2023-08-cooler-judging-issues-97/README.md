# Original link
https://github.com/sherlock-audit/2023-08-cooler-judging/issues/97
0xbepresent

high

# Lender can block loan repayments using malicious `Callback` contract causing the collateral lost
## Summary

The lender can block the loan repayment using malicious `Callback` contract causing the borrower to lost the collateral.

## Vulnerability Detail

The [Cooler::repayLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L151C14-L151C23) function helps to repay the loan and get the collateral back. Additionally, when a loan request is accepted [the lender can implement](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L263) a [CollateralCallback](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/CoolerCallback.sol#L8) contract.

The problem is that when there is a repayment using the `repayLoan()` function, the function will call the `lender callback` (code line 185).

```solidity
File: Cooler.sol
151:     function repayLoan(uint256 loanID_, uint256 repaid_) external returns (uint256) {
...
183: 
184:         // If necessary, trigger lender callback.
185:         if (loan.callback) CoolerCallback(loan.lender).onRepay(loanID_, repaid_);
...
```

Malicious lender can implement a malicious `CoolerCallback::onRepay()` function which consumes all the gas forwarded to it exhausting all the available gas causing to be unable to repay the loan.

## Impact

The borrower will lost the collateral.

## Code Snippet

- [Cooler::repayLoan()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L151C14-L151C23) function.

## Tool used

Manual review

## Recommendation

Implements a fixed gas consumption to the lender's `CoolerCallback` contract.

Duplicate of #187
