# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/289
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L19
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L124
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L168


# Vulnerability details

## Impact

The [GovNFT.sol::maxBridge](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L19) variable is not used in the "bridging process". The [lzReceive()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L168) function can run out of gas if there are many Tokens passing to another chain.

The GovNFTs can get caught in a chain because they can be burned in chain A but in Chain B the [lzReceive](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L168) can not process the mint.

## Proof of Concept

[GovNFT.sol::maxBridge](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L19) limit is not used in [crossChain()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFT.sol#L124)


## Tools used
VsCode

## Recommended Mitigation Steps

Add a limit of GovNFT before sending the NFTs to another chain.