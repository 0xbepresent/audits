# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/521
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L41-L43


# Vulnerability details

## Impact
When more than 10 mulitsig, it is impossible to modify or delete the old ones, making it impossible to create new valid ones.

## Proof of Concept

MultisigManager limits the number of Multisig to 10, which cannot be deleted or replaced after they have been disable
This will have a problem, if the subsequent use of 10, all 10 for some reason, be disabled
Then it is impossible to add new ones and replace the old ones, so you have to continue using the old Multisig at risk

```solidity
    function registerMultisig(address addr) external onlyGuardian {
        int256 multisigIndex = getIndexOf(addr);
        if (multisigIndex != -1) {
            revert MultisigAlreadyRegistered();
        }
        uint256 index = getUint(keccak256("multisig.count"));
        if (index >= MULTISIG_LIMIT) {
            revert MultisigLimitReached(); //***@audit limt 10, and no other way to delete or replace the old Multisig ***//
        }
```

## Tools Used

## Recommended Mitigation Steps

add replace old mulitsig method

```solidity
    function replaceMultisig(address addr,address oldAddr) external onlyGuardian {
        int256 multisigIndex = getIndexOf(oldAddr);
        if (multisigIndex == -1) {
            revert MultisigNotFound();
        }

        setAddress(keccak256(abi.encodePacked("multisig.item", multisigIndex, ".address")), addr);
        emit RegisteredMultisig(addr, msg.sender);
    }
```