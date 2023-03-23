# Original link
https://github.com/code-423n4/2022-09-vtvl-findings/issues/331
1 - Public functions that are never called from within the contracts should be declared external to save gas
==

Insatance of this issue:

```
contracts/VTVLVesting.sol#L398       function withdrawAdmin(uint112 _amountRequested) public onlyAdmin {    
```

2 - Unused imports
==

Import of unnecessary files costs deployment gas

```
contracts/VTVLVesting.sol#L6         import "@openzeppelin/contracts/access/Ownable.sol";
contracts/AccessProtected.sol#L5     import "@openzeppelin/contracts/access/Ownable.sol";
```

3 - Multiple require statements instead of &&
==

Require statements including conditions with the && operator can be broken down in multiple require statements to save gas.

```
contracts/VTVLVesting.sol#L344

require(_startTimestamps.length == length &&
        _endTimestamps.length == length &&
        _cliffReleaseTimestamps.length == length &&
        _releaseIntervalsSecs.length == length &&
        _linearVestAmounts.length == length &&
        _cliffAmounts.length == length, 
        "ARRAY_LENGTH_MISMATCH"
);
```

4 - Unnecessary checked arithmetic in for loop
==

There is no risk that the loop counter can overflow, using solidity's unchecked block saves gas.

Instance of this issue:

```
contracts/VTVLVesting.sol#L353      for (uint256 i = 0; i < length; i++) {
```

Unchecked implementation example:

```
for (uint256 i; i < 10;) {
    j++;
    unchecked {
        ++i;
    }
}
```

5 - ```x =+ y``` costs more gas than ```x = x + y``` for state variables
==

Instance of this issue:

```
contracts/VTVLVesting.sol#L302        numTokensReservedForVesting += allocatedAmount; // track the allocated amount
```

6 - Amount is not validated
==

Implement the required statement that amount should be greater than 0 in order to save gas.

Instances of this issue:

```
contracts/token/VariableSupplyERC20Token.sol#L36             function mint(address account, uint256 amount) public onlyAdmin {
```

 