# Original link
https://github.com/code-423n4/2023-01-drips-findings/issues/163
# Lines of code

https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L104-L119


# Vulnerability details

## Impact

The `Caller` contract enables users to authorize other users to execute tx on their behalf.
This option enables the authorized/delegated user to add more users to the authorized users list.
In case the original user is trying to remove an authorized user (i.e. run `unauthorize()`), the delegated user can simply front run that tx to add another user, then after `unauthorized()` is executed the delegated can use the added user to gain his authority back.

This would allow the malicious delegated user to keep executing txs on behalf of the original user and cause them a loss of funds (e.g. collecting funds on their behalf and sending it to the attacker's address).

## Proof of Concept

The test below demonstrates such a scenario:
* Alice authorizes Bob
* Bob becomes malicious and Alice now wants to remove him
* Bob noticed the tx to unauthorize him and front runs it by authorizing Eve
* Alice `unauthorize()` tx is executed
* Bob now authorizes himself back again via Eve's account


Front running can be done either by sending a tx with a higher gas price (usually tx are ordered in a block by the gas price / total fee), or by paying additional fee to the validator if they manage to run their tx without reverting (i.e. by sending additional ETH to `block.coinbase`, hoping validator will notice it).
It's true that Alice can run `unauthorize()` again and again and needs to succeed only once, but:
* Bob can keep adding other users and Alice would have to keep removing them all
* This is an ongoing battle that can last forever, and Alice might not have enough knowledge, resources and time to deal with it right away. This might take hours or days, and in the meanwhile Alice might be receiving a significant amount of drips that would be stolen by Bob.



```diff
diff --git a/test/Caller.t.sol b/test/Caller.t.sol
index 861b351..3e4be22 100644
--- a/test/Caller.t.sol
+++ b/test/Caller.t.sol
@@ -125,6 +125,24 @@ contract CallerTest is Test {
         vm.expectRevert(ERROR_UNAUTHORIZED);
         caller.callAs(sender, address(target), data);
     }
+    
+    function testFrontRunUnauthorize() public {
+        bytes memory data = abi.encodeWithSelector(target.run.selector, 1);
+        address bob = address(0xbab);
+        address eve = address(0xefe);
+        // Bob is authorized by Alice
+        authorize(sender, bob);
+
+        // Bob became malicious and Alice now wants to remove him
+        // Bob sees the `unauthorize()` call and front runs it with authorizing Eve
+        authorizeCallAs(sender, bob, eve);
+
+        unauthorize(sender, bob);
+
+        // Eve can now give Bob his authority back
+        authorizeCallAs(sender, eve, bob);
+
+    }
 
     function testAuthorizingAuthorizedReverts() public {
         authorize(sender, address(this));
@@ -257,6 +275,13 @@ contract CallerTest is Test {
         assertTrue(caller.isAuthorized(authorizing, authorized), "Authorization failed");
     }
 
+    function authorizeCallAs(address originalUser,address delegated, address authorized) internal {
+        bytes memory data = abi.encodeWithSelector(Caller.authorize.selector, authorized);
+        vm.prank(delegated);
+        caller.callAs(originalUser, address(caller),  data);
+        assertTrue(caller.isAuthorized(originalUser, authorized), "Authorization failed");
+    }
+
     function unauthorize(address authorizing, address unauthorized) internal {
         vm.prank(authorizing);
         caller.unauthorize(unauthorized);

```

## Recommended Mitigation Steps
Don't allow authorized users to call `authorize()` on behalf of the original user.
This can be done by replacing `_msgSender()` with `msg.sender` at `authorize()`, if the devs want to enable authorizing by signing I think the best way would be to add a dedicated function for that (other ways to prevent calling `authorize()` from `callAs()` can increase gas cost for normal calls, esp. since we'll need to cover all edge cases of recursive calls with `callBatched()`).