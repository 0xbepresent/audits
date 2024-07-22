# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/377
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L110
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L122


# Vulnerability details

## Impact

selfdestruct() will not be available after EIP-4758. This EIP will rename the SELFDESTRUCT opcode and replace its functionality. It will no longer destroy code or storage, so, the contract still will be available. 
In this case it will break the logic of the project because it will not work as aspected:

FixedPrice.sol
- After call the [cancel() function](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L50) the contract still will be available. Users will be able to buy a number of NFTs even if the current sale is cancelled. 

OpenEdition.sol
- After call the [cancel() function](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L75) the contract still will be available. Users will be able to buy a number of NFTs even if the current sale is cancelled. 

## Proof of Concept

According to [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758):
- The SELFDESTRUCT opcode is renamed to SENDALL, and now only immediately moves all ETH in the account to the target; it no longer destroys code or storage or alters the nonce.
- All refunds related to SELFDESTRUCT are removed.

## Tools Used

Manual Review

## Recommended Mitigation Steps

The architecture should be changed to avoid that problem.
