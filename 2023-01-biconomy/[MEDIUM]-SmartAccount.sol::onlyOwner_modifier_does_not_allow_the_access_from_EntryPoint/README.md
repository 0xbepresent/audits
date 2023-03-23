# Original link
https://github.com/code-423n4/2023-01-biconomy-findings/issues/106
# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L460
https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L465


# Vulnerability details

## Impact

The ```SmartAccount.sol::execute()``` and ```SmartAccount.sol::executeBatch()``` does not allow the access from the EntryPoint. The line [461](https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L461) and [466](https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L466) calls the ```_requireFromEntryPointOrOwner``` function which indicates that the function could be called by the EntryPoint or the owner.

Those functions are important for the smart account management so it important for the protocol to call them from the EntryPoint.

## Proof of Concept

As you can see in the code, the ```onlyOwner``` modifier is called before the ```_requireFromEntryPointOrOwner``` function so the EntryPoint can not call those functions.

```solidity
File: SmartAccount.sol
460:     function execute(address dest, uint value, bytes calldata func) external onlyOwner{
461:         _requireFromEntryPointOrOwner();
462:         _call(dest, value, func);
463:     }
464: 
465:     function executeBatch(address[] calldata dest, bytes[] calldata func) external onlyOwner{
466:         _requireFromEntryPointOrOwner();
467:         require(dest.length == func.length, "wrong array lengths");
468:         for (uint i = 0; i < dest.length;) {
469:             _call(dest[i], 0, func[i]);
470:             unchecked {
471:                 ++i;
472:             }
473:         }
474:     }

```

## Tools used

VsCode

## Recommended Mitigation Steps

Removes the ```onlyOwner``` modifier and the ```_requireFromEntryPointOrOwner()``` function will check if the sender is the Owner or the EntryPoint.
