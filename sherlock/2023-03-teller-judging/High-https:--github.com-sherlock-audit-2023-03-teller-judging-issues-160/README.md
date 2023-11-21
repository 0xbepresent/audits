# Original link
https://github.com/sherlock-audit/2023-03-teller-judging/issues/160
0xbepresent

high

# Malicious lender can assign his own commitment to another victim lender

## Summary

In the [LenderCommitmentForwarder](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol) contract. A malicious lender's commitment can be assigned to another victim lender via the [updateCommitment()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L208) function causing the victim lender to have unauthorized commitments and unautorized loans from borrowers who takes those unauthorized commitments.


## Vulnerability Detail

The [LenderCommitmentForwarder](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol) contract helps to lender to make a capital commitment so the borrower can get an instant loan.

The problem is that an malicious lender can assign his own commitment to another victim lender via the [updateCommitment()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L208) function. The ```updateCommitment()``` does not validate the lender change.

## Impact

Please see the next scenario:

1. An attacker see the [commitments](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L57) and get a lender who has a commitment of X principal token.
2. The attacker creates a malicious commitment for him with a low interest rate for the same principal token. Then he assign it to the victim lender.
3. The attacker [accept the commitment](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L300) (he also acts as a borrower) and the victim lender now has an unauthorized loan.

I created a test where the malicious lender (attacker) commitment is assigned to a victim lender via the ```updateCommitment()``` function causing the victim lender to have unauthorized commitments. Test steps:

1. Attacker creates his commitment via [createCommitment()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L177) function.
2. Attacker assigns his own commitment to a victim lender via the [updateCommitment()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L208) function.
3. Now the commitment, that belonged to the attacker, is assigned to a victim lender

```solidity
File: LenderCommitmentForwarder_Test.sol
275:     function test_attacker_can_assign_his_own_commitment_to_another_lender() public {
276:         // An attacker can assign his own commitment to another lender (victim)
277:         // 1. Attacker creates his commitment
278:         // 2. Attacker assigns his own commitment to a victim lender
279:         // 3. Now the commitment, that belonged to the attacker, is assigned to a victim lender
280:         //
281:         // 1. Attacker creates his commitment
282:         //
283:         address attacker = address(1337);
284:         address victimLender = address(712713);
285:         LenderCommitmentForwarder.Commitment
286:             memory c = LenderCommitmentForwarder.Commitment({
287:                 maxPrincipal: maxPrincipal,
288:                 expiration: expiration,
289:                 maxDuration: maxDuration,
290:                 minInterestRate: minInterestRate,
291:                 collateralTokenAddress: address(collateralToken),
292:                 collateralTokenId: collateralTokenId,
293:                 maxPrincipalPerCollateralAmount: maxPrincipalPerCollateralAmount,
294:                 collateralTokenType: collateralTokenType,
295:                 lender: attacker,
296:                 marketId: marketId,
297:                 principalTokenAddress: address(principalToken)
298:             });
299:         lenderCommitmentForwarder.setCommitment(0, c);
300:         //
301:         // 2. Attacker assigns his own commitment to a victim lender
302:         //
303:         c.lender = victimLender;
304:         vm.prank(attacker);
305:         lenderCommitmentForwarder.updateCommitment(0, c);
306:         //
307:         // 3. Now the commitment, that belonged to the attacker, is assigned to a victim lender
308:         //
309:         assertEq(
310:             lenderCommitmentForwarder.getCommitmentLender(0),
311:             victimLender
312:         );
313:     }
```

## Code Snippet

The [updateCommitment()](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/LenderCommitmentForwarder.sol#L208) function does not validate the lender change.

```solidity
File: LenderCommitmentForwarder.sol
208:     function updateCommitment(
209:         uint256 _commitmentId,
210:         Commitment calldata _commitment
211:     ) public commitmentLender(_commitmentId) {
212:         require(
213:             _commitment.principalTokenAddress ==
214:                 commitments[_commitmentId].principalTokenAddress,
215:             "Principal token address cannot be updated."
216:         );
217:         require(
218:             _commitment.marketId == commitments[_commitmentId].marketId,
219:             "Market Id cannot be updated."
220:         );
221: 
222:         commitments[_commitmentId] = _commitment;
223: 
224:         validateCommitment(commitments[_commitmentId]);
225: 
226:         emit UpdatedCommitment(
227:             _commitmentId,
228:             _commitment.lender,
229:             _commitment.marketId,
230:             _commitment.principalTokenAddress,
231:             _commitment.maxPrincipal
232:         );
233:     }
```

## Tool used

Manual review

## Recommendation

Add a lender validation in the ```updateCommitment()``` function. The lender must not change in the ```updateCommitment()``` function.

Duplicate of #260
