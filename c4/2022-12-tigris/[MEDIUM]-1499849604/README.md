# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/334
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/GovNFT.sol#L19-L20


# Vulnerability details

## Impact
In GovNFT, setMaxBridge function is provided to set maxBridge, but this variable is not used, literally it should be used to limit the number of GovNFTs crossing chain, but it doesn't work in GovNFT.
```solidity
    uint256 public maxBridge = 20;
...
    function setMaxBridge(uint256 _max) external onlyOwner {
        maxBridge = _max;
    }
```
## Proof of Concept
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/GovNFT.sol#L19-L20
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/GovNFT.sol#L311-L313
## Tools Used
None
## Recommended Mitigation Steps
Consider applying the maxBridge variable