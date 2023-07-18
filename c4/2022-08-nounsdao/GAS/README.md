# Original link
https://github.com/code-423n4/2022-08-nounsdao-findings/issues/109
1 - Internal functions only called once can be inlined to save gas
==

Not inlining costs more gas because of two extra ```JUMP``` instructions and additional stack operations needed for function calls.

There are 3 instances of this issue:

```
contracts/governance/NounsDAOLogicV2.sol#L305    function queueOrRevertInternal(
contracts/governance/NounsDAOLogicV2.sol#L866    function proposalCreationBlock(Proposal storage proposal) internal view returns (uint256) {
contracts/governance/NounsDAOLogicV2.sol#L974    function _refundGas(uint256 startGas) internal {
```

2 - Process functions only if they have not been processed
==

In addition to checking that the function is not in the "Executed" state, you can also check that the function has not been processed.

Example:

```
# cancel function
require(state(proposalId) != ProposalState.Canceled, 'NounsDAO::cancel: proposal already canceled');

# veto function
require(state(proposalId) != ProposalState.Vetoed, 'NounsDAO::veto: proposal already vetoed');
```

There are 2 instances of this issue:

```
contracts/governance/NounsDAOLogicV2.sol#L347    require(state(proposalId) != ProposalState.Executed, 'NounsDAO::cancel: cannot cancel executed proposal');
contracts/governance/NounsDAOLogicV2.sol#L377    require(state(proposalId) != ProposalState.Executed, 'NounsDAO::veto: cannot veto executed proposal');
```

3 - Unnecessary checked arithmetic in for loop
==

There is no risk that the loop counter can overflow, using solidity's unchecked block saves gas.

There are 4 instances of this issue:

```
governance/NounsDAOLogicV2.sol#292 => for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#331 => for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#359 => for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#385 => for (uint256 i = 0; i < proposal.targets.length; i++) {
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

4 - Cache Array Length Outside of Loop
==

Caching the array length outside a loop saves reading it on each iteration, as long as the array's length is not changed during the loop.

There are 4 instances of this issue:

```
governance/NounsDAOLogicV2.sol#292    for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#331    for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#359    for (uint256 i = 0; i < proposal.targets.length; i++) {
governance/NounsDAOLogicV2.sol#385    for (uint256 i = 0; i < proposal.targets.length; i++) {
```

5 - Consider using custom errors
==

The contracts in scope use Solidity >=0.8.4, consider using Custom Errors as they are more gas efficient while allowing developers to describe the error in detail using [Natspec](https://docs.soliditylang.org/en/latest/natspec-format.html)
