# Original link
https://github.com/code-423n4/2022-09-vtvl-findings/issues/3
# Lines of code

https://github.com/code-423n4/2022-09-vtvl/blob/main/contracts/token/VariableSupplyERC20Token.sol#L36-L46


# Vulnerability details

## Impact
The admin of the token is not constrained to minting `maxSupply_`, they can mint any number of tokens.

## Proof of Concept
```js
// If we're using maxSupply, we need to make sure we respect it
// mintableSupply = 0 means mint at will
if(mintableSupply > 0) {
	require(amount <= mintableSupply, "INVALID_AMOUNT");
	// We need to reduce the amount only if we're using the limit, if not just leave it be
	mintableSupply -= amount;
}
```
The logic is as follows: if the amount that can be minted is zero, treat this as an infinite mint. Else require that the minted amount is not larger than mintable supply.

One can note that it is possible to mint all mintable supply. Then the mintable supply will be `0` which will be interpreted as infinity and any number of tokens will be possible to be minted.

## Tools Used
Manual analysis

## Recommended Mitigation Steps
Treat `2 ** 256 - 1` as infinity instead of `0`.
