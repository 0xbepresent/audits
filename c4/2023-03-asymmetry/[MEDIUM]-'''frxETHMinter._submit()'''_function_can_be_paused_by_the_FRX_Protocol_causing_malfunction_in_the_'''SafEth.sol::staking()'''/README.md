# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/553
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L101
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L63


# Vulnerability details

## Impact

The [SfrxEth.sol::deposit()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L94) is used by [SafEth.sol::stake()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L91) to be able to deposit the ```user's ETH```. The users deposits are distributed among the ```Derivatives``` based on the weight assigned to each ```derivative```. The ```SfrxEth.sol::deposit()``` uses the [frxETHMinterContract.submitAndDeposit()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#LL101C9-L101C80) function to be able to mint ```frxETH``` and deposit it to receive ```sfrxETH``` in one transaction. 

The problem is that the [frxETHMinter._submit()](https://github.com/FraxFinance/frxETH-public/blob/7f7731dbc93154131aba6e741b6116da05b25662/src/frxETHMinter.sol#L85) function can [be paused](https://github.com/FraxFinance/frxETH-public/blob/7f7731dbc93154131aba6e741b6116da05b25662/src/frxETHMinter.sol#L87) by the FRX Protocol causing a revert in all the staking process. If for some reasons the [frxMinter pauses their submits](https://github.com/FraxFinance/frxETH-public/blob/7f7731dbc93154131aba6e741b6116da05b25662/src/frxETHMinter.sol#L182) all the users who wants to use ```SafEth.sol::stake()``` will be reverted causing a bad experience in the protocol because the ```stake()``` function will not be available for any other healthy ```Derivative```.

## Proof of Concept

Please see the next scenario:
1. User stakes his ```ETH``` via [stake()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L91) function.
2. Some amount of ```ETH``` is going to the [SfrxEth.deposit()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L94) function.
3. The [frxETHMinterContract.submitAndDeposit{}())](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L101) is called.
4. Then in the frxMinterContract the [_submit()](https://github.com/FraxFinance/frxETH-public/blob/7f7731dbc93154131aba6e741b6116da05b25662/src/frxETHMinter.sol#L72) function is called.
5. The ```_submit()``` function is reverted because [it is paused](https://github.com/FraxFinance/frxETH-public/blob/7f7731dbc93154131aba6e741b6116da05b25662/src/frxETHMinter.sol#L87).
6. All the transaction is reverted causing the staking to other ```Derivatives``` are not possible.

## Tools used

VScode

## Recommended Mitigation Steps

If the FRX Derivative has weight assigned and for some reason the FRX protocol is paused, it would be possible to maintain the ETH in the ```SafEth.sol``` contract until an admin adjust the weights and rebalance all the ```ETH``` to healthy derivatives. This prevents the stake function from not being available due to paused/unhealthy derivatives.