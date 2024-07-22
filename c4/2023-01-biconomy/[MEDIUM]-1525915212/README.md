# Original link
https://github.com/code-423n4/2023-01-biconomy-findings/issues/390
# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/main/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L460-L461
https://github.com/code-423n4/2023-01-biconomy/blob/main/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L465-L466


# Vulnerability details

## Description
`execute` and `executeBatch` in `SmartAccount.sol` can only be called by owner, not EntryPoint:

```javascript
File: SmartAccount.sol

460:    function execute(address dest, uint value, bytes calldata func) external onlyOwner{
461:        _requireFromEntryPointOrOwner();
462:        _call(dest, value, func);
463:    }
464:
465:    function executeBatch(address[] calldata dest, bytes[] calldata func) external onlyOwner{
466:        _requireFromEntryPointOrOwner();
467:        require(dest.length == func.length, "wrong array lengths");
468:        for (uint i = 0; i < dest.length;) {
469:            _call(dest[i], 0, func[i]);
470:            unchecked {
471:                ++i;
472:            }
473:        }
474:    }
```

From [EIP-4337](https://eips.ethereum.org/EIPS/eip-4337):

> - **Call the account with the `UserOperation`’s calldata.** It’s up to the account to choose how to parse the calldata; an expected workflow is for the account to have an execute function that parses the remaining calldata as a series of one or more calls that the account should make.

## Impact
This breaks the interaction with EntryPoint

## Proof of Concept
The reference implementation has both these functions without any onlyOwner modifiers:

https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/samples/SimpleAccount.sol#L56-L73
```javascript
56:    /**
57:     * execute a transaction (called directly from owner, not by entryPoint)
58:     */
59:    function execute(address dest, uint256 value, bytes calldata func) external {
60:        _requireFromEntryPointOrOwner();
61:        _call(dest, value, func);
62:    }
63:
64:    /**
65:     * execute a sequence of transaction
66:     */
67:    function executeBatch(address[] calldata dest, bytes[] calldata func) external {
68:        _requireFromEntryPointOrOwner();
69:        require(dest.length == func.length, "wrong array lengths");
70:        for (uint256 i = 0; i < dest.length; i++) {
71:            _call(dest[i], 0, func[i]);
72:        }
73:    }
```

## Tools Used
vscode

## Recommended Mitigation Steps
Remove `onlyOwner` modifier from `execute` and `executeBatch`