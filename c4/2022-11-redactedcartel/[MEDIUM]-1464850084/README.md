# Original link
https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/137
# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/main/src/vaults/AutoPxGlp.sol#L210
https://github.com/code-423n4/2022-11-redactedcartel/blob/main/src/PirexGmx.sol#L497-L516


# Vulnerability details

## Impact
Compounded assets may be lost because `AutoPxGlp::compound` can be called by anyone and minimum amount of Glp and USDG are under caller's control. The only check concerning minValues is that they are not zero (1 will work, however from the perspective of real tokens e.g. 1e6, or 1e18 it's virtually zero). Additionally, internal smart contract functions use it as well with minimal possible value (e.g. `beforeDeposit` function)

## Proof of Concept
`compound` function calls PirexGmx::depositGlp, that uses external GMX reward router to mint and stake GLP.
https://snowtrace.io/address/0x82147c5a7e850ea4e28155df107f2590fd4ba327#code
```solidity
141:     function mintAndStakeGlpETH(uint256 _minUsdg, uint256 _minGlp) external payable nonReentrant returns (uint256) {
    ...
148: uint256 glpAmount = IGlpManager(glpManager).addLiquidityForAccount(address(this), account, weth, msg.value, _minUsdg, _minGlp);
```
Next `GlpManager::addLiquidityForAccount` is called
https://github.com/gmx-io/gmx-contracts/blob/master/contracts/core/GlpManager.sol#L103
```
    function addLiquidityForAccount(address _fundingAccount, address _account, address _token, uint256 _amount, uint256 _minUsdg, uint256 _minGlp) external override nonReentrant returns (uint256) {
        _validateHandler();
        return _addLiquidity(_fundingAccount, _account, _token, _amount, _minUsdg, _minGlp);
    }
```

which in turn uses vault to swap token for specific amount of USDG before adding liquidity:
https://github.com/gmx-io/gmx-contracts/blob/master/contracts/core/GlpManager.sol#L217

The amount of USGD to mint is calcualted by GMX own price feed:
https://github.com/gmx-io/gmx-contracts/blob/master/contracts/core/Vault.sol#L765-L767

In times of market turbulence, or price oracle  manipulation, all compound value may be lost

## Tools Used
VS Code,  arbiscan.io

## Recommended Mitigation Steps
Don't depend on user passing minimum amounts of usdg and glp tokens. Use GMX oracle to get current price, and additionally check it against some other price feed (e.g. ChainLink)