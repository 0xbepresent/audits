# Original link
https://github.com/sherlock-audit/2023-04-hubble-exchange-judging/issues/95
0xbepresent

high

# A malicious `vUSD` withdrawal receiver can cause a DOS in the `vUSD.processWithdrawals()` function

## Summary

A malicious `vUSD` withdrawal receiver can cause a DOS in the `vUSD.processWithdrawals()` function causing all withdrawals to get stuck.

## Vulnerability Detail

The [VUSD._withdrawTo()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L109) function helps to queue the withdrawals in the [withdrawals](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L112C9-L112C20) list, then the [processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L65C14-L65C32) can be called in order to send all the withdrawals to their [corresponding receiver](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L75).

The problem is [low-level external calls](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L75) can exhaust all available gas. The `withdrawal` receiver can be a contract which executes arbitrary logic and receives/spend all the available gas, then the `processWithdrawals()` may not have enough gas to complete the execution, causing a denial of service. Consider the next scenario:

1. Malicious actor mints some `vUSD` via [mintWithReserve()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L45) function.
2. Malicious actor creates a malicious receiver smart contract that consumes all the gas forwarded to it.
3. Malicious actor calls the [withdrawTo()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L58C14-L58C24) function using his Malicious smart contract as a receiver (parammeter `to`).
4. Others users create more withdrawals
5. When the [processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L65C14-L65C32) is called, the [Malicious smart contract will be called](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L75) exhausting all the available gas causing `processWithdrawals()` may not have enough gas to complete the execution.
6. All the next withdrawals will be get stuck in the vUSD contract.

## Impact

The `vUSD` withdrawals will be get stuck causing a complete damage for the legitimate users who wants their `usdc`. Since the malicious actor can create withdrawals of `5e6` each one, the attack can be executed many times causing excessive resource consumption and a degraded quality of service.

## Code Snippet

The [processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L65C14-L65C32) and [VUSD._withdrawTo()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L109) functions:

```solidity
File: VUSD.sol
065:     function processWithdrawals() external override whenNotPaused nonReentrant {
066:         uint reserve = address(this).balance;
067:         require(reserve >= withdrawals[start].amount, 'Cannot process withdrawals at this time: Not enough balance');
068:         uint i = start;
069:         while (i < withdrawals.length && (i - start) < maxWithdrawalProcesses) {
070:             Withdrawal memory withdrawal = withdrawals[i];
071:             if (reserve < withdrawal.amount) {
072:                 break;
073:             }
074: 
075:             (bool success, bytes memory data) = withdrawal.usr.call{value: withdrawal.amount}("");
076:             if (success) {
077:                 reserve -= withdrawal.amount;
078:             } else {
079:                 emit WithdrawalFailed(withdrawal.usr, withdrawal.amount, data);
080:             }
081:             i += 1;
082:         }
083:         // re-entracy not possible, hence can update `start` at the end
084:         start = i;
085:     }
...
...
109:     function _withdrawTo(address to, uint amount) internal {
110:         require(amount >= 5 * (10 ** PRECISION), "min withdraw is 5 vusd");
111:         burn(amount); // burn vusd from msg.sender
112:         withdrawals.push(Withdrawal(to, amount * SCALING_FACTOR));
113:     }
```

## Tool used

Manual review

## Recommendation

Validates that the withdrawal receiver is not a contract.

Duplicate of #116 
