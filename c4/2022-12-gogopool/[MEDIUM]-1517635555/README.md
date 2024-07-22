# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/702
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L80-L91


# Vulnerability details


For every created minipool a multisig address is set to continue validator interactions.

Every minipool multisig address get assigned by calling `requireNextActiveMultisig`.

This function always return the first enabled multisig address.

In case the specific address is disabled all created minipools will be stuck with this address which increase the probability of also funds being stuck.

## Impact
Probability of funds being stuck increases if `requireNextActiveMultisig` always return the same address.

## Proof of Concept
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L80-L91

## Tools Used
Manual review

## Recommended Mitigation Steps
Use a strategy like [round robin](https://en.wikipedia.org/wiki/Round-robin_item_allocation) to assign next active multisig to minipool

Something like this :

```solidity
private uint nextMultisigAddressIdx;

function requireNextActiveMultisig() external view returns (address) {
    uint256 total = getUint(keccak256("multisig.count"));
    address addr;
    bool enabled;

    uint256 i = nextMultisigAddressIdx; // cache last used
    if (nextMultisigAddressIdx==total) {
        i = 0;
    }

    for (; i < total; i++) {
        (addr, enabled) = getMultisig(i);
        if (enabled) {
            nextMultisigAddressIdx = i+1;
            return addr;
        }
    }
    
    revert NoEnabledMultisigFound();
}
```