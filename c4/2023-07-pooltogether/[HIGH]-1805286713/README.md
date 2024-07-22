# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/396
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L394-L402


# Vulnerability details

## Impact
The `Vault.mintYieldFee` external function is used to mint `Vault shares` to the yield fee `_recipient`. The function is an external function and can be called by anyone since there is no access control. The function will revert only under following two conditions:

1. if the Vault is undercollateralized
2. if the `_shares` are greater than the accrued `_yieldFeeTotalSupply`.

The issue with this function is it allows the caller to set the `_recipient` (Address of the yield fee recipient). It does not use the `_yieldFeeRecipient` state variable which was set in the `Vault.constructor` as the `yield fee recipient`.

Which means any one can steal the available `yield fee` from this vault (As long as above two revert conditions are not satisfied) by `minting shares` to his own address or to any address of his choice. 

## Proof of Concept

```solidity
  function mintYieldFee(uint256 _shares, address _recipient) external {
    _requireVaultCollateralized();
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_recipient, _shares);

    emit MintYieldFee(msg.sender, _recipient, _shares);
  }
```

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L394-L402

## Tools Used
Manual Review and VSCode

## Recommended Mitigation Steps

Hence it is recommended to use the `_yieldFeeRecipient` state variable value as the `yield fee recipient` inside the `Vault.mintYieldFee` external function and to remove the input parameter `address _recipient` from the `Vault.mintYieldFee` function. So that the caller will not be able to mint shares to any arbitory address of his choice and steal the yield fee of the protocol.

The updated function should be as follows:

```solidity
  function mintYieldFee(uint256 _shares) external {
    _requireVaultCollateralized();
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_yieldFeeRecipient, _shares);

    emit MintYieldFee(msg.sender, _recipient, _shares);
  } 
```


## Assessed type

Other