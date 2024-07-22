# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/465
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L653
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1053
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1068


# Vulnerability details

## Impact
The setHooks function in Vault.sol allows users to set arbitrary hooks, potentially enabling them to make external calls with unintended consequences. This vulnerability could lead to various unexpected behaviors, such as unauthorized side transactions with gas paid unbeknownst to the claimer, reentrant calls, or denial-of-service attacks on claiming transactions.

[Vault.sol#L653](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L653)
```solidity
function setHooks(VaultHooks memory hooks) external {
    _hooks[msg.sender] = hooks; 

    emit SetHooks(msg.sender, hooks);
  }
```

## Proof of Concept
Consider the following side contract and malicious hook implementation:

Side Contract:
```solidity
// SPDX-License_Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/console.sol";

contract SideContract {
  uint256 public codex;

  function execute() external {
    codex = 1;
    console.log("Entered side contract");
  }
}
```

Malicious Hook:
```solidity
// SPDX-License_Identifier: MIT
pragma solidity 0.8.17;

import { SideContract } from "./SideContract.sol";

contract ViciousHook {
  SideContract public sideContract = new SideContract();

  function beforeClaimPrize(
    address winner,
    uint8 tier,
    uint32 prizeIndex
  ) external returns (address) {
    sideContract.execute();

    return address(this);
  }
}
```

Modified Test File:
```diff
diff --git a/test/unit/Vault/Vault.t.sol b/test/unit/Vault/Vault.t.sol
index d0d6d30..65caca1 100644
--- a/test/unit/Vault/Vault.t.sol
+++ b/test/unit/Vault/Vault.t.sol
@@ -4,6 +4,7 @@ pragma solidity 0.8.17;
 import { UnitBaseSetup, LiquidationPair, PrizePool, TwabController, VaultMock, ERC20, IERC20, IERC4626 } from "test/utils/UnitBaseSetup.t.sol";
 import { IVaultHooks, VaultHooks } from "../../../src/interfaces/IVaultHooks.sol";
 import "src/Vault.sol";
+import { ViciousHook } from "./ViciousHook.sol";
 
 contract VaultTest is UnitBaseSetup {
   /* ============ Events ============ */
@@ -174,6 +175,25 @@ contract VaultTest is UnitBaseSetup {
     vm.stopPrank();
   }
 
+  function testClaimPrize_viciousHook() public {
+    ViciousHook viciousHook = new ViciousHook();
+    vm.startPrank(alice);
+    VaultHooks memory hooks = VaultHooks({
+      useBeforeClaimPrize: true,
+      useAfterClaimPrize: false,
+      implementation: IVaultHooks(address(viciousHook))
+    });
+    vault.setHooks(hooks);
+    vm.stopPrank();
+
+    vm.startPrank(address(claimer));
+
+    mockPrizePoolClaimPrize(uint8(1), alice, 0, address(viciousHook), 1e18, address(claimer));
+    claimPrize(uint8(1), alice, 0, 1e18, address(claimer));
+
+    vm.stopPrank();
+  }
+
   function testClaimPrize_beforeHook() public {
     vm.startPrank(alice);
     VaultHooks memory hooks = VaultHooks({
```
When running the test with `forge test --match-test testClaimPrize_viciousHook -vv ` the output is:
```bash
Running 1 test for test/unit/Vault/Vault.t.sol:VaultTest
[PASS] testClaimPrize_viciousHook() (gas: 321545)
Logs:
  Entered side contract

Test result: ok. 1 passed; 0 failed; finished in 7.29ms
```
This indicates that it is possible for a hook to make an external call and modify the EVM state. With that fact, attack vectors are multiple.

## Tools Used
Foundry

## Recommended Mitigation Steps
To prevent any malicious calls there are two possible solutions:
- Change the `IVaultHook.sol` to set the hooks as view functions and prevent EVM state changes:
```diff
diff --git a/src/interfaces/IVaultHooks.sol b/src/interfaces/IVaultHooks.sol
index 15217b8..95482c1 100644
--- a/src/interfaces/IVaultHooks.sol
+++ b/src/interfaces/IVaultHooks.sol
@@ -23,7 +23,7 @@ interface IVaultHooks {
     address winner,
     uint8 tier,
     uint32 prizeIndex
-  ) external returns (address);
+  ) external view returns (address);
 
   /// @notice Triggered after the prize pool claim prize function is called.
   /// @param winner The user who won the prize and for whom this hook is attached
@@ -37,5 +37,5 @@ interface IVaultHooks {
     uint32 prizeIndex,
     uint256 payout,
     address recipient
-  ) external;
+  ) external view;
 }

```
- As this solution may disturb the business logic, another solution would be to cap the gas used by the hooks. In `Vault.sol`, set a gas limit variable that can be adjusted by the owner of the vault for flexibility:
[Vault.sol#L1053](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1053)
[Vault.sol#L1068](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1068)
```solidity
  function _claimPrize(
    address _winner,
    uint8 _tier,
    uint32 _prizeIndex,
    uint96 _fee,
    address _feeRecipient
  ) internal returns (uint256) {
    VaultHooks memory hooks = _hooks[_winner];
    address recipient;
    if (hooks.useBeforeClaimPrize) {
      recipient = hooks.implementation.beforeClaimPrize{gas: gasLimit}(_winner, _tier, _prizeIndex); @audit-info
    } else {
      recipient = _winner;
    }
...

if (hooks.useAfterClaimPrize) {
      hooks.implementation.afterClaimPrize{gas: gasLimit}( //@audit-info
        _winner,
        _tier,
        _prizeIndex,
        prizeTotal - _fee,
        recipient
      );
    }

    return prizeTotal;
  }

```



## Assessed type

Other