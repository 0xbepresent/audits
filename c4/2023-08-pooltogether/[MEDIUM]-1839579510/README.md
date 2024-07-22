# Original link
https://github.com/code-423n4/2023-08-pooltogether-findings/issues/92
# Lines of code

https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L170


# Vulnerability details

## Impact
If the recipient maliciously enters the blacklist of `priceToken`, it may cause `rngComplete()` to fail to execute successfully

## Proof of Concept
The current implementation of `RngRelayAuction.rngComplete()` immediately transfers the `prizeToken` to the `recipient`

```solidity
  function rngComplete(
    uint256 _randomNumber,
    uint256 _rngCompletedAt,
    address _rewardRecipient,
    uint32 _sequenceId,
    AuctionResult calldata _rngAuctionResult
  ) external returns (bytes32) {
...
    for (uint8 i = 0; i < _rewards.length; i++) {
      uint104 _reward = uint104(_rewards[i]);
      if (_reward > 0) {
@>      prizePool.withdrawReserve(auctionResults[i].recipient, _reward);
        emit AuctionRewardDistributed(_sequenceId, auctionResults[i].recipient, i, _reward);
      }
    }  
..

```solidity
contract PrizePool is TieredLiquidityDistributor {
  function withdrawReserve(address _to, uint104 _amount) external onlyDrawManager {
    if (_amount > _reserve) {
      revert InsufficientReserve(_amount, _reserve);
    }
    _reserve -= _amount;
@>  _transfer(_to, _amount);
    emit WithdrawReserve(_to, _amount);
  }

  function _transfer(address _to, uint256 _amount) internal {
    _totalWithdrawn += _amount;
@>  prizeToken.safeTransfer(_to, _amount);
  }

```

There is a risk that if `prizeToken` is a `token` with a blacklisting mechanism, such as :`USDC`. 
then `recipient` can block `rngComplete()` by maliciously entering the `USDC` blacklist. 
Since `RngAuctionRelayer` supports `AddressRemapper`, users can more simply specify blacklisted addresses via `remapTo()`


## Tools Used

## Recommended Mitigation Steps

Add `claims` mechanism, `rngComplete()` only record `cliamable[token][user]+=rewards`
Users themselves go to `claims`


## Assessed type

Context