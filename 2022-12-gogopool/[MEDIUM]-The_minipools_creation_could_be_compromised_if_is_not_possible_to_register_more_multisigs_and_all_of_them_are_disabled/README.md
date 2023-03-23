# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/349
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L35
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L55
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L68


# Vulnerability details

## Impact

The ```MultisigManager.sol``` allows to register the multisigs that are valid for the Minipools administration. There is a [limit for the registration](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L41).

If for some reason all the Multisig are disabled/compromised, it would not possible to add more multisig because the limit. The protocol can not create/enable more multisigs and is not possible to create minipools because there are not any enabled multisig.

All multisigs could be compromised and arises the necessity to register more multisigs in order to assign them for the minipool administration. It is an edge case but it is possible.

## Proof of Concept

I did the next test:

1. Register and enable multisigs in order to reach the limit
2. Disable all the multisigs
3. Register another multisig and the transaction will be reverted because the registration reach the limit
4. It is not possible to create more multisigs and the protocol runs out of valid multisigs

```solidity
function testMultisigLimitAfterDisableAllOfThem() public {
    // Unable to assign more multisig when all of them are disabled
    // 1. Register and enable multisigs and reach the limit
    // 2. Disable all the multisigs
    // 3. Register another multisig and the transaction will be reverted because
    // 	  the registration reach the limit
    uint256 count = multisigMgr.getCount();
    uint256 limit = multisigMgr.MULTISIG_LIMIT();
    //
    // 1. Register multisigs and reach the limit
    //
    vm.startPrank(guardian);
    for (uint256 i = 0; i < limit - count; i++) {
        address randomMultisig = randAddress();
        multisigMgr.registerMultisig(randomMultisig);
        multisigMgr.enableMultisig(randomMultisig);
    }
    assertEq(multisigMgr.getCount(), limit);
    vm.expectRevert(MultisigManager.MultisigLimitReached.selector);
    multisigMgr.registerMultisig(randAddress());
    //
    // 2. Disable all the multisigs
    //
    for (uint256 i = 0; i < limit - count; i++) {
        (address multisigAddress,) = multisigMgr.getMultisig(i);
        multisigMgr.disableMultisig(multisigAddress);
    }
    //
    // 3. Register another multisig and the transaction will be reverted because
    // the registration reach the limit
    //
    address randomMultisig = randAddress();
    vm.expectRevert(MultisigManager.MultisigLimitReached.selector);
    multisigMgr.registerMultisig(randomMultisig);
    vm.stopPrank();
}
```

## Tools used

Foundry/VsCode

## Recommended Mitigation Steps

Count only the validated/enabled multisigs in order to control the limit.