# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/142
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L63-L101
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L228-L245
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170-L183
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/WstEth.sol#L56-L66
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L108-L128
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L74-L75


# Vulnerability details

## Impact

rETH derivative can be bought through uniswap if the deposit contract is not open.

While a maxSlippage variable is set, the price of rETH on uniswap is the spot price and is only determined during the transaction opening sandwich opportunity for MEV researchers as long as the slippage stays below maxSlippage.

This is also true for WstETH (on withdraw) and frxETH (on deposit and withdraw) that go through a curve pool when unstaking (and staking for frxETH). While the curve pool has much more liquidity and the assumed price is a 1 - 1 ratio for WstETH and frxETH seem to be using a twap price before applying the slippage, these attacks are less likely to happen so I will only describe rETH.

## Proof of Concept

While the current rETH derivative contract uses uniswapv3 0,05% pool, I'll be using the uniswapv2 formula (https://amm-calculator.vercel.app/) to make this example simplier, in both case sandwiching is possible.

Default slippage is set to 1% on rETH contract at deployment. (see: https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L44)

Let's take a pool of 10,000 rETH for 10,695 eth (same ratio is on the univ3 0,05% pool on the 26th of march).

User wants to stake 100ETH, a third of it will be staked through rETH according to a 1/3 weight for each derivative.

Bundle:

TX1: 

Researcher swap 100 ETH in for 92.63 rETH
new pool balance: 9907.36 rETH - 10795 ETH

TX2: 

User stake his ETH, the rocketPool deposit contract is close so the deposit function takes the current spot price of the pool and then applies 1% slippage to it to get minOut.

Current ratio: eth = 0.9177 rETH
ETH to swap for reth: 33.3333~
So minOut -> 33.3333 * 0.9177 * 0.99 = 30.284 rETH

(see: https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170-L183)

Contract swap 33.3333 ETH for 30.498 rETH (slippage of 0.61% so below 1% and received more than minOut)

New pool balance: 9876.86 rETH - 10828.33 ETH

TX3:

Researcher swap back the 92.63 rETH in for 100.61~ ETH
new pool balance: 9969.49 rETH - 10727.72 ETH

Researcher made 0.61~ ETH of profit, could be more as we only applied a 0,61% slippage but we can go as far as 1% in the current rETH contract.

Univ3 pool would could even worse as Researcher with a lot of liquidity could be able to drain one side (liquidity is very concentrated), add liquidity in a tight range execute the stake and then remove liquidity and swap back.

## Tools Used

Manual review + https://amm-calculator.vercel.app/

## Recommended Mitigation Steps

The rETH price should be determined using the TWAP price and users should be able to input minOut in the stake, unstake and rebalanceToWeight function.