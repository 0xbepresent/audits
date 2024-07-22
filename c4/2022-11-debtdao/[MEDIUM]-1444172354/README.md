# Original link
https://github.com/code-423n4/2022-11-debtdao-findings/issues/355
# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/utils/LineLib.sol#L59-L74


# Vulnerability details

## Impact

If ERC20 and eth are transferred at same time, the mistakenly sent eth will be locked.
There are several functions could be affected and cause user fund lock:
- `addCollateral()`
- `addCredit()`
- `increaseCredit()`
- `depositAndClose()`
- `depositAndRepay()`
- `close()`

## Proof of Concept

In `receiveTokenOrETH()`, different logic is used to handle ERC20 and eth transfer. However, in the ERC20 if block, mistakenly sent eth will be ignored. This part of eth will be locked in the contract.
```solidity
// Line-of-Credit/contracts/utils/LineLib.sol
    function receiveTokenOrETH(
      address token,
      address sender,
      uint256 amount
    )
      external
      returns (bool)
    {
        if(token == address(0)) { revert TransferFailed(); }
        if(token != Denominations.ETH) { // ERC20
            IERC20(token).safeTransferFrom(sender, address(this), amount);
        } else { // ETH
            if(msg.value < amount) { revert TransferFailed(); }
        }
        return true;
    }
```


## Tools Used
Manual analysis.

## Recommended Mitigation Steps

In the ERC20 part, add check for `msg.value` to ensure no eth is sent:
```solidity
        if(token != Denominations.ETH) { // ERC20
            if (msg.value > 0) { revert TransferFailed(); }
            IERC20(token).safeTransferFrom(sender, address(this), amount);
        } else { // ETH
```
