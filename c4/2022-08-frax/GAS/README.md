# Original link
https://github.com/code-423n4/2022-08-frax-findings/issues/114

1 - Using private rather than public for constants save gas.
==

There are 3 instances of this issue:

```
./FraxlendPairDeployer.sol:49:    uint256 public DEFAULT_MAX_LTV = 75000; // 75% with 1e5 precision
./FraxlendPairDeployer.sol:50:    uint256 public GLOBAL_MAX_LTV = 1e8; // 1000x (100,000%) with 1e5 precision, protects from rounding errors in LTV calc
./FraxlendPairDeployer.sol:51:    uint256 public DEFAULT_LIQ_FEE = 10000; // 10% with 1e5 precision
```

Gas Report using [this test](https://github.com/0xbepresent/gas-lab/blob/main/examples/PrivateConstants.t.sol):

```
╭────────────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/PrivateConstants.t.sol:Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞════════════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                                    ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 49987                                              ┆ 175             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                      ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withPublicConstant                                 ┆ 2247            ┆ 2247 ┆ 2247   ┆ 2247 ┆ 1       │
╰────────────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭────────────────────────────────────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ src/test/PrivateConstants.t.sol:Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════════════════════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost                                    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 24075                                              ┆ 149             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withPrivateConstant                                ┆ 146             ┆ 146 ┆ 146    ┆ 146 ┆ 1       │
╰────────────────────────────────────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
```

2 - Usign calldata instead of memory for read-only arguments in external functions save gas
==

When a function with a memory array is called externally, the abi.decode() step has to use a for-loop to copy each index of the calldata to the memory index. Using calldata directly, obliviates the need for such a loop in the contract code and runtime execution.

There are 4 instances of this issue:

```
./FraxlendPairDeployer.sol#310:    function deploy(bytes memory _configData) external returns (address _pairAddress) {
./FraxlendPairDeployer.sol#357:    bytes memory _configData,
./FraxlendPairDeployer.sol#362:    address[] memory _approvedBorrowers,
./FraxlendPairDeployer.sol#363:    address[] memory _approvedLenders
./FraxlendPairDeployer.sol#398:    function globalPause(address[] memory _addresses) external returns (address[] memory _updatedAddresses) {
./FraxlendPairCore.sol#1067:       address[] memory _path
```

Gas report using [this test](https://github.com/0xbepresent/gas-lab/blob/main/examples/CallDataExternal.t.sol)
```
╭────────────────────────────────────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ src/test/CallDataExternal.t.sol:Contract0 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════════════════════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost                                    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 109359                                             ┆ 578             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withMemoryParam                                    ┆ 914             ┆ 914 ┆ 914    ┆ 914 ┆ 1       │
╰────────────────────────────────────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────────────────────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ src/test/CallDataExternal.t.sol:Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════════════════════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost                                    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 82329                                              ┆ 443             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withCallDataParam                                  ┆ 815             ┆ 815 ┆ 815    ┆ 815 ┆ 1       │
╰────────────────────────────────────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
```

3 - ```x =+ y``` costs more gas than ```x = x + y``` for state variables
==

There is 2 instance of this issue:

```
./FraxlendPairCore.sol#L773: totalCollateral += _collateralAmount;
./FraxlendPairCore.sol#L815: totalCollateral -= _collateralAmount;
```

Gas report example using [this test](https://github.com/0xbepresent/gas-lab/blob/main/examples/EqualState.t.sol)

```
╭──────────────────────────────────────────────┬─────────────────┬───────┬────────┬───────┬─────────╮
│ src/test/EqualState.t.sol:Contract0 contract ┆                 ┆       ┆        ┆       ┆         │
╞══════════════════════════════════════════════╪═════════════════╪═══════╪════════╪═══════╪═════════╡
│ Deployment Cost                              ┆ Deployment Size ┆       ┆        ┆       ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 38487                                        ┆ 223             ┆       ┆        ┆       ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                ┆ min             ┆ avg   ┆ median ┆ max   ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withEqualPlusState                           ┆ 22317           ┆ 22317 ┆ 22317  ┆ 22317 ┆ 1       │
╰──────────────────────────────────────────────┴─────────────────┴───────┴────────┴───────┴─────────╯
╭──────────────────────────────────────────────┬─────────────────┬───────┬────────┬───────┬─────────╮
│ src/test/EqualState.t.sol:Contract1 contract ┆                 ┆       ┆        ┆       ┆         │
╞══════════════════════════════════════════════╪═════════════════╪═══════╪════════╪═══════╪═════════╡
│ Deployment Cost                              ┆ Deployment Size ┆       ┆        ┆       ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37287                                        ┆ 217             ┆       ┆        ┆       ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                ┆ min             ┆ avg   ┆ median ┆ max   ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withOutEqualPlusState                        ┆ 22298           ┆ 22298 ┆ 22298  ┆ 22298 ┆ 1       │
╰──────────────────────────────────────────────┴─────────────────┴───────┴────────┴───────┴─────────╯
```

4 - Unnecessary checked arithmetic in for loop
==

There is no risk that the loop counter can overflow, using solidity's unchecked block saves gas.

There are 8 instances of this issue:

```
./FraxlendPair.sol:289:            for (uint256 i = 0; i < _lenders.length; i++) {
./FraxlendPair.sol:308:            for (uint256 i = 0; i < _borrowers.length; i++) {
./libraries/SafeERC20.sol:27:      for (i = 0; i < 32 && data[i] != 0; i++) {
./FraxlendWhitelist.sol:51:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendWhitelist.sol:66:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendWhitelist.sol:81:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendPairCore.sol:265:        for (uint256 i = 0; i < _approvedBorrowers.length; ++i) {
./FraxlendPairCore.sol:270:        for (uint256 i = 0; i < _approvedLenders.length; ++i) {
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

Gas report example using [this test](https://github.com/0xbepresent/gas-lab/blob/main/examples/Unchecked.t.sol):

```
╭─────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/Unchecked.t.sol:Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞═════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                             ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 55105                                       ┆ 307             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                               ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withOutUnChecked                            ┆ 2068            ┆ 2068 ┆ 2068   ┆ 2068 ┆ 1       │
╰─────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭─────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/Unchecked.t.sol:Contract1 contract ┆                 ┆      ┆        ┆      ┆         │
╞═════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                             ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 53705                                       ┆ 300             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                               ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withUnchecked                               ┆ 1408            ┆ 1408 ┆ 1408   ┆ 1408 ┆ 1       │
╰─────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
```

5 - Cache array length outside of loop
==

Caching the array length outside a loop saves reading it on each iteration, as long as the array's length is not changed during the loop.

There are 7 instances of this issue:

```
./FraxlendPair.sol:289:            for (uint256 i = 0; i < _lenders.length; i++) {
./FraxlendPair.sol:308:            for (uint256 i = 0; i < _borrowers.length; i++) {
./FraxlendWhitelist.sol:51:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendWhitelist.sol:66:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendWhitelist.sol:81:        for (uint256 i = 0; i < _addresses.length; i++) {
./FraxlendPairCore.sol:265:        for (uint256 i = 0; i < _approvedBorrowers.length; ++i) {
./FraxlendPairCore.sol:270:        for (uint256 i = 0; i < _approvedLenders.length; ++i) {
```

Gas Report using [this test](https://github.com/0xbepresent/gas-lab/blob/main/examples/ArrayLength.t.sol):

```
╭───────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/ArrayLength.t.sol:Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞═══════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                               ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 137640                                        ┆ 412             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                 ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withArrayLengthInside                         ┆ 3108            ┆ 3108 ┆ 3108   ┆ 3108 ┆ 1       │
╰───────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭───────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/ArrayLength.t.sol:Contract1 contract ┆                 ┆      ┆        ┆      ┆         │
╞═══════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                               ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 137840                                        ┆ 413             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                 ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withArrayLengthOutside                        ┆ 2813            ┆ 2813 ┆ 2813   ┆ 2813 ┆ 1       │
╰───────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
```