# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/188
# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L314-L335
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L479-L493


# Vulnerability details

## Impact
When users withdraw their vault tokens PublicVault mint WithdrawProxy's shares token for them and at the end of the epoch PublicVault would calculated WithdrawProxy's assets and update PublicVault assets and start the next epoch. if a lot of users withdraws their funds then the value of the `totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)` (the amount belongs to the WithdrawProxy) would be higher than `yIntercept` and code would revert because of the underflow when setting the new value of the `yIntercept`. This would cause last users to not be able to withdraw their funds and contract epoch system to be broken for a while.

## Proof of Concept
This is part of `processEpoch()` code that calculates ratio between WithdrawProxy and PublicVault:
```
  function processEpoch() public {
.....
.....
    // reset liquidationWithdrawRatio to prepare for re calcualtion
    s.liquidationWithdrawRatio = 0;

    // check if there are LPs withdrawing this epoch
    if ((address(currentWithdrawProxy) != address(0))) {
      uint256 proxySupply = currentWithdrawProxy.totalSupply();

      s.liquidationWithdrawRatio = proxySupply
        .mulDivDown(1e18, totalSupply())
        .safeCastTo88();

      currentWithdrawProxy.setWithdrawRatio(s.liquidationWithdrawRatio);
      uint256 expected = currentWithdrawProxy.getExpected();

      unchecked {
        if (totalAssets() > expected) {
          s.withdrawReserve = (totalAssets() - expected)
            .mulWadDown(s.liquidationWithdrawRatio)
            .safeCastTo88();
        } else {
          s.withdrawReserve = 0;
        }
      }
      _setYIntercept(
        s,
        s.yIntercept -
          totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)
      );
      // burn the tokens of the LPs withdrawing
      _burn(address(this), proxySupply);
    }
```
As you can see in the line `_setYIntercept(s, s.yIntercept - totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18))` code tries to set new value for `yIntercept` but This is `totalAssets()` code:
```
  function totalAssets()
    public
    view
    virtual
    override(ERC4626Cloned)
    returns (uint256)
  {
    VaultData storage s = _loadStorageSlot();
    return _totalAssets(s);
  }

  function _totalAssets(VaultData storage s) internal view returns (uint256) {
    uint256 delta_t = block.timestamp - s.last;
    return uint256(s.slope).mulDivDown(delta_t, 1) + uint256(s.yIntercept);
  }
```
So as you can see `totalAssets()` can be higher than `yIntercept` and if most of the user withdraw their funds(for example the last user) then the value of `liquidationWithdrawRatio` would be near `1` too and the value of ` totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)` would be bigger than `yIntercept` and call to `processEpoch()` would revert and code can't start the next epoch and user withdraw process can't be finished and funds would stuck in the contract.

## Tools Used
VIM

## Recommended Mitigation Steps
prevent underflow by calling `accrue()` in the begining of the `processEpoch()`