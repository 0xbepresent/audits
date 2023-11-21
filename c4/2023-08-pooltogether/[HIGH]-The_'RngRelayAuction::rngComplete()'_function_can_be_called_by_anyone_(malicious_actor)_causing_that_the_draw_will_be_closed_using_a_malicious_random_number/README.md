# Original link
https://github.com/code-423n4/2023-08-pooltogether-findings/issues/120
# Lines of code

https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L131
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L154
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L170


# Vulnerability details

## Impact

The [rngComplete()](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L131) function is called by the [RngAuctionRelayerDirect contract](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerDirect.sol#L39) or the [RngAuctionRelayerRemoteOwner](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerRemoteOwner.sol#L62) contract. The `rngComplete()` function receives the [RNG results](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/abstract/RngAuctionRelayer.sol#L33) from the rng auction. 

The problem here is that the `rngComplete()` function can be called by anyone not only the `relayers`. This is very dangerous because malicious actors can close draws without any restriction using their own malicious [randomNumber](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L154). Additionally the malicious actor can [receive the rewards](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L170).

## Proof of Concept

The [rngComplete()](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L131) function does not have any restriction about who can call it so in the `code line 154` the `prizePool` will [close the draw](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L154) using a [custom number](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L132) specified in the `code line 132` by the malicious actor.

```solidity
File: RngRelayAuction.sol
131:   function rngComplete(
132:     uint256 _randomNumber,
133:     uint256 _rngCompletedAt,
134:     address _rewardRecipient,
135:     uint32 _sequenceId,
136:     AuctionResult calldata _rngAuctionResult
137:   ) external returns (bytes32) {
138:     if (_sequenceHasCompleted(_sequenceId)) revert SequenceAlreadyCompleted();
139:     uint64 _auctionElapsedSeconds = uint64(block.timestamp < _rngCompletedAt ? 0 : block.timestamp - _rngCompletedAt);
140:     if (_auctionElapsedSeconds > (_auctionDurationSeconds-1)) revert AuctionExpired();
141:     // Calculate the reward fraction and set the draw auction results
142:     UD2x18 rewardFraction = _fractionalReward(_auctionElapsedSeconds);
143:     _auctionResults.rewardFraction = rewardFraction;
144:     _auctionResults.recipient = _rewardRecipient;
145:     _lastSequenceId = _sequenceId;
146: 
147:     AuctionResult[] memory auctionResults = new AuctionResult[](2);
148:     auctionResults[0] = _rngAuctionResult;
149:     auctionResults[1] = AuctionResult({
150:       rewardFraction: rewardFraction,
151:       recipient: _rewardRecipient
152:     });
153: 
154:     uint32 drawId = prizePool.closeDraw(_randomNumber);
155: 
156:     uint256 futureReserve = prizePool.reserve() + prizePool.reserveForOpenDraw();
157:     uint256[] memory _rewards = RewardLib.rewards(auctionResults, futureReserve);
158: 
159:     emit RngSequenceCompleted(
160:       _sequenceId,
161:       drawId,
162:       _rewardRecipient,
163:       _auctionElapsedSeconds,
164:       rewardFraction
165:     );
166: 
167:     for (uint8 i = 0; i < _rewards.length; i++) {
168:       uint104 _reward = uint104(_rewards[i]);
169:       if (_reward > 0) {
170:         prizePool.withdrawReserve(auctionResults[i].recipient, _reward);
171:         emit AuctionRewardDistributed(_sequenceId, auctionResults[i].recipient, i, _reward);
172:       }
173:     }
174: 
175:     return bytes32(uint(drawId));
176:   }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Restrict only the `relayers` can call the [rngComplete()](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L131) function.


## Assessed type

Access Control