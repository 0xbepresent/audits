# Original link
https://github.com/code-423n4/2022-08-fiatdao-findings/issues/44
1 - Internal functions only called once can be inlined to save gas
==

Not inlining costs more gas because of extra ```JUMP``` instructions and additional stack operations needed for function calls.

There are 2 instances of this issue:

```
./VotingEscrow.sol:662:    function _calculatePenaltyRate(uint256 end)
./VotingEscrow.sol:732:    function _findUserBlockEpoch(address _addr, uint256 _block)
```

https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L662

https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L732

2 - Unnecessary variable convertion and assigment
==

There is a value assigment to a ```delegated property``` https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L537 but in the line https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L540 the property is assigned to zero. The line VotingEscrow.sol#L537 could be omitted to save gas

There are 2 instances of this issue
```
https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L537
https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L642
```

3 - ```x =+ y``` costs more gas than ```x = x + y``` for state variables
==

There is 1 instance of this issue:

```
VotingEscrow.sol
L654: penaltyAccumulated += penaltyAmount;
```

4 - Unnecessary checked arithmetic in for loop
==

There is no risk that the loop counter can overflow, using solidity's unchecked block saves gas.

There are 4 instances of this issue:

```
./VotingEscrow.sol:309:  for (uint256 i = 0; i < 255; i++) {
./VotingEscrow.sol:717:  for (uint256 i = 0; i < 128; i++) {
./VotingEscrow.sol:739:  for (uint256 i = 0; i < 128; i++) {
./VotingEscrow.sol:834:  for (uint256 i = 0; i < 255; i++) {
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

Gas Report example using [this gist](https://gist.github.com/0xbepresent/707eefd3ead1b0a297b0f17d3dc54c7f):

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