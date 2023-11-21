# Original link
https://github.com/code-423n4/2022-09-vtvl-findings/issues/348
# Lines of code

https://github.com/code-423n4/2022-09-vtvl/blob/f68b7f3e61/contracts/AccessProtected.sol#L39


# Vulnerability details

## Impact
If a wrong/invalid address is given when calling setAdmin, some contract's functions could be lost access. While setAdmin checks for zero address, there is no validation of the new address being correct.

Also, there is no validation that the contract has at least one admin.

The design of the setAdmin function can block the contract's functions forever, so functions like "createClaim", "createClaimsBatch", "withdrawAdmin", "revokeClaim", "withdrawOtherToken", "setAdmin", "mint" would be locked.

```
contracts/AccessProtected.sol#L39    function setAdmin(address admin, bool isEnabled) public onlyAdmin {
```

## Proof of Concept

- Alice, the admin, executes setAdmin to an address which is accidentally invalid.
```
setAdmin(invalid_address, True)
```
- Alice, who is giving up being admin, will invalidate her own access.
```
setAdmin(alice_address, False)
```
- Now, nobody has access to critical functions.

## Tools used
VisualStudio Code

## Recommended Mitigation Steps
Change the single-step admin assignation to a two-step process where the current Admin first approves a new address as a pendingAdmin. That pendingAdmin has to then claim the ownership in a separate transaction. An incorrectly set pendingAdmin can be reset by changing it again to the correct one who can then successfully claim it in the second step.

Also, in the admin assignation process make sure that the contract has at least one admin.