# Original link
https://github.com/code-423n4/2022-09-quickswap-findings/issues/210
# Lines of code

https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraFactory.sol#L91-L95
https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L918
https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L546-L549


# Vulnerability details

Collected community fees from each swap and flash loan are immediately sent to the defined `AlgebraFactor.vaultAddress` address. Contrary to the used pull pattern in Uniswap V3.

Having fees (ERC-20 tokens) immediately sent to the vault within each swap and flash loan, opens up potential issues if the vault address is set to the zero address.

The following `AlgebraPool` functions are affected:

- `swap`,
- `swapSupportingFeeOnInputTokens`, and
- `flash`

## Impact

If the `AlgebraFactor.vaultAddress` address is set to `address(0)`, **all** Algebra pools deployed by this factory contract will have their swap and flash loan functionality affected in a either completely broken way (transactions will revert due to ERC-20 token transfers to the zero address) or fees are sent (effectively burned) to the zero address. In the end, it will depend on the ERC-20 token implementation if it reverts or simply burns the fees.

## Proof of Concept

[AlgebraFactory.setVaultAddress](https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraFactory.sol#L91-L95)

`AlgebraFactory.setVaultAddress` is used by the owner of the `AlgebraFactory` to set the vault address.

```solidity
function setVaultAddress(address _vaultAddress) external override onlyOwner {
  require(vaultAddress != _vaultAddress);
  emit VaultAddress(_vaultAddress);
  vaultAddress = _vaultAddress;
}
```

[AlgebraPool.sol#L918](https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L918)

```solidity
function flash(
  address recipient,
  uint256 amount0,
  uint256 amount1,
  bytes calldata data
) external override lock {
  [...]

  address vault = IAlgebraFactory(factory).vaultAddress();

  uint256 paid0 = balanceToken0();
  require(balance0Before.add(fee0) <= paid0, 'F0');
  paid0 -= balance0Before;

  if (paid0 > 0) {
    uint8 _communityFeeToken0 = globalState.communityFeeToken0;
    uint256 fees0;
    if (_communityFeeToken0 > 0) {
      fees0 = (paid0 * _communityFeeToken0) / Constants.COMMUNITY_FEE_DENOMINATOR;
      TransferHelper.safeTransfer(token0, vault, fees0); // @audit-info `vault` is used as the recipient of community fees
    }
    totalFeeGrowth0Token += FullMath.mulDiv(paid0 - fees0, Constants.Q128, _liquidity);
  }

  uint256 paid1 = balanceToken1();
  require(balance1Before.add(fee1) <= paid1, 'F1');
  paid1 -= balance1Before;

  if (paid1 > 0) {
    uint8 _communityFeeToken1 = globalState.communityFeeToken1;
    uint256 fees1;
    if (_communityFeeToken1 > 0) {
      fees1 = (paid1 * _communityFeeToken1) / Constants.COMMUNITY_FEE_DENOMINATOR;
      TransferHelper.safeTransfer(token1, vault, fees1); // @audit-info `vault` is used as the recipient of community fees
    }
    totalFeeGrowth1Token += FullMath.mulDiv(paid1 - fees1, Constants.Q128, _liquidity);
  }

  emit Flash(msg.sender, recipient, amount0, amount1, paid0, paid1);
}
```

[AlgebraPool.\_payCommunityFee](https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L546-L549)

This function is called from within the `AlgebraPool.swap` and `AlgebraPool.swapSupportingFeeOnInputTokens` functions.

```solidity
function _payCommunityFee(address token, uint256 amount) private {
  address vault = IAlgebraFactory(factory).vaultAddress();
  TransferHelper.safeTransfer(token, vault, amount);
}
```

## Tools Used

Manual review

## Recommended mitigation steps

Either

- consider adding a check if `vault == address(0)` in `AlgebraPool._payCommunityFee` and in `AlgebraPool.flash`,
- prevent setting `vault` to `address(0)` in `AlgebraFactory.setVaultAddress`, or
- use a pull pattern to collect community (protocol) fees
