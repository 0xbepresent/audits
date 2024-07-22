# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/206
# Lines of code

https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L596-L599
https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L648-L664


# Vulnerability details

## Impact
The default delegate value for a user is `address(0)` which maps to the user delegating to themselves. If a user had delegated to another address and wanted to reset their delegated balance back to themselves they would lose all of their funds contributed to the vault.

## Proof of Concept
As mentioned above, the default behaviour for a user is that they delegate their balance to themselves, where the actual default value in storage is the 0 address:

```
function _delegateOf(address _vault, address _user) internal view returns (address) {
    address _userDelegate;

    if (_user != address(0)) {
      _userDelegate = delegates[_vault][_user];

      // If the user has not delegated, then the user is the delegate
      if (_userDelegate == address(0)) {
        _userDelegate = _user;
      }
    }

    return _userDelegate;
  }
```

When a user wants to delegate their balance, they call `delegate` in `TwabController.sol` and specify which vault they want to delegate the balance of, and to which address they want to delegate to. This calls `_delegate` under the hood:

```
function _delegate(address _vault, address _from, address _to) internal {
    address _currentDelegate = _delegateOf(_vault, _from);
    if (_to == _currentDelegate) {
      revert SameDelegateAlreadySet(_to);
    }

    delegates[_vault][_from] = _to;

    _transferDelegateBalance(
      _vault,
      _currentDelegate,
      _to,
      uint96(userObservations[_vault][_from].details.balance)
    );

    emit Delegated(_vault, _from, _to);
  }
```

If a user wanted to reset the delegation to themselves they would specify `_to` as `address(0)`. However, the issue with this is that the underlying `_transferDelegateBalance` call will mistakenly move the delegated user funds to the 0 address.

At this point the user might try to call `delegate` again with their actual address, however now the `(_to == _currentDelegate)` check will be true and revert, because of the behaviour specified earlier. The user also can't delegate to any other address because they don't own their own delegate balance anymore. Their funds are officially lost forever.

Below is a change to the existing test suite that can be executed with `forge test -vvv --match-path test/unit/Vault/Withdraw.t.sol` to demonstrate this issue:

```
diff --git a/test/unit/Vault/Withdraw.t.sol b/test/unit/Vault/Withdraw.t.sol
index 6a15a59..3cec9e3 100644
--- a/test/unit/Vault/Withdraw.t.sol
+++ b/test/unit/Vault/Withdraw.t.sol
@@ -47,6 +47,36 @@ contract VaultWithdrawTest is UnitBaseSetup {
     vm.stopPrank();
   }
 
+  function testFundsLostForever() external {
+    vm.startPrank(alice);
+    uint256 _amount = 1000e18;
+    underlyingAsset.mint(alice, _amount);
+
+    // Alice deposits as usual
+    _deposit(underlyingAsset, vault, _amount, alice);
+
+    // Alice decides she wants to delegate to bob
+    twabController.delegate(address(vault), bob);
+
+    // Alice now tries to reset her delegation
+    twabController.delegate(address(vault), address(0));
+
+    // At this point the funds are lost!! Alice tries to recover her funds in any way...
+    
+    // Alice tries to delegate back to herself but can't
+    vm.expectRevert();
+    twabController.delegate(address(vault), alice);
+
+    // Alice also can't delegate to any other address
+    vm.expectRevert();
+    twabController.delegate(address(vault), bob);
+    
+    // Alice can't withdraw because her funds have been lost forever :(
+    // Expecting a revert with "DelegateBalanceLTAmount(0, 1000000000000000000000)"
+    vault.withdraw(vault.maxWithdraw(alice), alice, alice);
+    vm.stopPrank();
+  }
+
   function testWithdrawMoreThanMax() external {
     vm.startPrank(alice);
 
```


## Tools Used
Manual review + foundry

## Recommended Mitigation Steps
The simplest way to fix this issue is to prevent delegating back to the 0 address. If a user delegates away from the default, then they can delegate back to themselves by specifying their own address:

```
diff --git a/src/TwabController.sol b/src/TwabController.sol
index a7e2d51..ae7b9ea 100644
--- a/src/TwabController.sol
+++ b/src/TwabController.sol
@@ -646,6 +646,7 @@ contract TwabController {
    * @param _to the address to delegate to
    */
   function _delegate(address _vault, address _from, address _to) internal {
+    require(_to != address(0), "Cannot delegate back to 0 address");
     address _currentDelegate = _delegateOf(_vault, _from);
     if (_to == _currentDelegate) {
       revert SameDelegateAlreadySet(_to);

```


## Assessed type

Invalid Validation