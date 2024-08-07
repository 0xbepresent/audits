# Original link
https://github.com/code-423n4/2023-04-caviar-findings/issues/858
# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L731-L738


# Vulnerability details

## Impact

Private pools have a "change" fee setting that is used to charge fees when a change is executed in the pool (user swaps tokens for some tokens in the pool). This setting is controlled by the `changeFee` variable, which is intended to be defined using 4 decimals of precision:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L87-L88

```solidity
87:     /// @notice The change/flash fee to 4 decimals of precision. For example, 0.0025 ETH = 25. 500 USDC = 5_000_000.
88:     uint56 public changeFee;
```

As the comment says, in the case of ETH a value of 25 should represent 0.0025 ETH. In the case of an ERC20 this should be scaled accordingly based on the number of decimals of the token. The implementation is defined in the `changeFeeQuote` function.

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L731-L738

```solidity
731:     function changeFeeQuote(uint256 inputAmount) public view returns (uint256 feeAmount, uint256 protocolFeeAmount) {
732:         // multiply the changeFee to get the fee per NFT (4 decimals of accuracy)
733:         uint256 exponent = baseToken == address(0) ? 18 - 4 : ERC20(baseToken).decimals() - 4;
734:         uint256 feePerNft = changeFee * 10 ** exponent;
735: 
736:         feeAmount = inputAmount * feePerNft / 1e18;
737:         protocolFeeAmount = feeAmount * Factory(factory).protocolFeeRate() / 10_000;
738:     }
```

As we can see in the previous snippet, in case the `baseToken` is an ERC20, then the exponent is calculated as `ERC20(baseToken).decimals() - 4`. The main issue here is that if the token decimals are less than 4, then the subtraction will cause an underflow due to Solidity's default checked math, causing the whole transaction to be reverted.

Such tokens with low decimals exist, one major example is [GUSD](https://etherscan.io/token/0x056Fd409E1d7A124BD7017459dFEa2F387b6d5Cd), Gemini dollar, which has only two decimals. If any of these tokens is used as the base token of a pool, then any call to the `change` will be reverted, as the scaling of the charge fee will result in an underflow.

## Proof of Concept

In the following test we recreate the "Gemini dollar" token (GUSD) which has 2 decimals and create a Private Pool using it as the base token. Any call to `change` or `changeFeeQuote` will be reverted due to an underflow error.

Note: the snippet shows only the relevant code for the test. Full test file can be found [here](https://gist.github.com/romeroadrian/06238839330315780b90d9202042ea0f).

```solidity
function test_PrivatePool_changeFeeQuote_LowDecimalToken() public {
    // Create a pool with GUSD which has 2 decimals
    ERC20 gusd = new GUSD();

    PrivatePool privatePool = new PrivatePool(
        address(factory),
        address(royaltyRegistry),
        address(stolenNftOracle)
    );
    privatePool.initialize(
        address(gusd), // address _baseToken,
        address(milady), // address _nft,
        100e18, // uint128 _virtualBaseTokenReserves,
        10e18, // uint128 _virtualNftReserves,
        500, // uint56 _changeFee,
        100, // uint16 _feeRate,
        bytes32(0), // bytes32 _merkleRoot,
        false, // bool _useStolenNftOracle,
        false // bool _payRoyalties
    );

    // The following will fail due an overflow. Calls to `change` function will always revert.
    vm.expectRevert();
    privatePool.changeFeeQuote(1e18);
}
```

## Recommendation

The implementation of `changeFeeQuote` should check if the token decimals are less than 4 and handle this case by dividing by the exponent difference to correctly scale it (i.e. `chargeFee / (10 ** (4 - decimals))`). For example, in the case of GUSD with 2 decimals, a `chargeFee` value of `5000` should be treated as `0.50`.
