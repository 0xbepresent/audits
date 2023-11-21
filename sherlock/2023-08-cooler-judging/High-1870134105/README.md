# Original link
https://github.com/sherlock-audit/2023-08-cooler-judging/issues/104
0xbepresent

medium

# `CoolerCallback` is not validated when there is a transfer ownership
## Summary

The lender can transfer the owner ship to another lender. The problem is that there are not validations that the new lender has implemented the `CoolerCallback`.

## Vulnerability Detail

When lender accepts a loan request, he can specify a callback, so the [lender must implements](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L233) `CoolerCallback abstract` (code line 241):

```solidity
File: Cooler.sol
233:     function clearRequest(
234:         uint256 reqID_,
235:         bool repayDirect_,
236:         bool isCallback_
237:     ) external returns (uint256 loanID) {
...
...
240:         // If necessary, ensure lender implements the CoolerCallback abstract.
241:         if (isCallback_ && !CoolerCallback(msg.sender).isCoolerCallback()) revert NotCoolerCallback();
...
...
```

Additionally, the lender can approve transfer to a new lender using the [approveTransfer()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L338) function, then the new lender accept the ownership using the [transferOwnership()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L347C14-L347C31) function.

The problem is that the `CoolerCallback().isCollerCallback()` is not validated for the new lender.

## Impact

If the loan has activated the `callback` and the new lender does not implement `CoolerCallback::isCollerCallback()` the loan will be completly broken:
- The loan repay will be reverted because the [CoolerCallback::onRepay()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L185) will not be implemented.
- The rollLoan() will be reverted because the [CoolerCallback::onRoll()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L216) will not be implemented.
- The lender can not claim default because the [CoolerCallback::onDefault()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L331) will not be implemented.

## Code Snippet

The [transferOwnership()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L347C14-L347C31) function:

```solidity
File: Cooler.sol
347:     function transferOwnership(uint256 loanID_) external {
348:         if (msg.sender != approvals[loanID_]) revert OnlyApproved();
349: 
350:         // Update the load lender.
351:         loans[loanID_].lender = msg.sender;
352:         // Clear transfer approvals.
353:         approvals[loanID_] = address(0);
354:     }
```

## Tool used

Manual review

## Recommendation

Implement a validation that the new lender should implement the `CoolerCallback::isCoolerCallback()`:

```diff
    function transferOwnership(uint256 loanID_) external {
        if (msg.sender != approvals[loanID_]) revert OnlyApproved();
++      if (loans[loanId_].callback && !CoolerCallback(msg.sender).isCoolerCallback()) revert NotCoolerCallback();

        // Update the load lender.
        loans[loanID_].lender = msg.sender;
        // Clear transfer approvals.
        approvals[loanID_] = address(0);
    }
```

Duplicate of #187
