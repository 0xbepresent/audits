# Original link
https://github.com/sherlock-audit/2023-06-unstoppable-judging/issues/104
0xbepresent

medium

# The `Vault._to_usd_oracle_price()` function uses the same `ORACLE_FRESHNESS_THRESHOLD` for all token prices feeds which is incorrect

## Summary

The same `ORACLE_FRESHNESS_THRESHOLD` is used for all the token prices feeds which can be dangerous because different pairs of tokens have different freshness intervals.

## Vulnerability Detail

The [ORACLE_FRESHNESS_THRESHOLD](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L55) is 24 hours constant. It is used to check if the [Oracle price is fresh](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L580) in the 580 code line.

```python
File: Vault.vy
562: def _to_usd_oracle_price(_token: address) -> uint256:
...
...
576:     round_id, answer, started_at, updated_at, answered_in_round = ChainlinkOracle(
577:         self.to_usd_oracle[_token]
578:     ).latestRoundData()
579: 
580:     assert (block.timestamp - updated_at) < ORACLE_FRESHNESS_THRESHOLD, "oracle not fresh"
...
...
```

The problem is that different pairs have diferrent `heartbeats`. For example the [LINK/USD](https://data.chain.link/arbitrum/mainnet/crypto-usd/link-usd) has a `heartbeat` of 3600 seconds so since the `ORACLE_FRESHNESS_THRESHOLD` is set to `24 hours`, the check for `LINK/USD` is useless since the its hearbeat is 3600 seconds. The same behaivour in [CRV/USD](https://data.chain.link/arbitrum/mainnet/crypto-usd/crv-usd) which has a heartbeat of 3600 seconds.

## Impact

Using the same `ORACLE_FRESHNESS_THRESHOLD` (heartbeat) for all the price feeds is not correct becuase the freshness validation would be useless for some pairs which can return stale data.

## Code Snippet

The [Vault._to_usd_oracle_price()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L562) function:

```python
File: Vault.vy
562: def _to_usd_oracle_price(_token: address) -> uint256:
563:     """
564:     @notice
565:         Retrieves the latest Chainlink oracle price for _token.
566:         Ensures that the Arbitrum sequencer is up and running and
567:         that the Chainlink feed is fresh.
568:     """
569:     assert self._sequencer_up(), "sequencer down"
570: 
571:     round_id: uint80 = 0
572:     answer: int256 = 0
573:     started_at: uint256 = 0
574:     updated_at: uint256 = 0
575:     answered_in_round: uint80 = 0
576:     round_id, answer, started_at, updated_at, answered_in_round = ChainlinkOracle(
577:         self.to_usd_oracle[_token]
578:     ).latestRoundData()
579: 
580:     assert (block.timestamp - updated_at) < ORACLE_FRESHNESS_THRESHOLD, "oracle not fresh"
581: 
582:     usd_price: uint256 = convert(answer, uint256)  # 8 dec
583:     return usd_price
```

## Tool used

Manual review

## Recommendation

Use the corresponding heartbeat `ORACLE_FRESHNESS_THRESHOLD` for each token in the `Vault._to_usd_oracle_price()` function.

