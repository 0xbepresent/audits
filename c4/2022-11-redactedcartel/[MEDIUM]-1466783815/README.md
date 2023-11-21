# Original link
https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/348
# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L242
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L227
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L321
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L345


# Vulnerability details

## Impact

The ```compound()``` function helps to swap the gmxBaseReward (ether) for GMX tokens then the GMX tokens are deposited for pxGMX. In the swap function the ```amountOutMinimum``` helps to put the minimum value that is expected for the swap. The [fixed amountOutMInimum](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol) assumes that the price can not be manipulated by different techniques so it is better to calculate the minimum accepted amount.

The ```amountOutMinimum``` should be calculated with the Uniswap SDK or an onchain price oracle as their [documentation](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps) says:

```
For a real deployment, this value should be calculated using our SDK or an onchain price oracle - this helps protect against getting an unusually bad price for a trade due to a front running sandwich or another type of price manipulation.
```

## Proof of Concept

The following lines use a fixed amountOutMinimum value:
- [src/vaults/AutoPxGmx.sol#L227](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L227)
- [src/vaults/AutoPxGmx.sol#L321](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L321)
- [src/vaults/AutoPxGmx.sol#L345](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L345)

## Tools used
VisualStudio

## Recommended Mitigation Steps

Calculates the ```amountOutMinimum``` using the Uniswap SDK, an onchain price oracle or a time weighted average price.