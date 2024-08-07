# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/601
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L219-L229
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81-L87


# Vulnerability details

After completing a task in the context of a quest, a user receives a signed hash that needs to be redeemed on-chain for a receipt that can later be claimed for a reward.

The receipt is minted in the `mintReceipt` function present in the `QuestFactory` contract:

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L219-L229

```solidity
function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
    if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
    if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
    if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
    if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();

    quests[questId_].addressMinted[msg.sender] = true;
    quests[questId_].numberMinted++;
    emit ReceiptMinted(msg.sender, questId_);
    rabbitholeReceiptContract.mint(msg.sender, questId_);
}
```

This function doesn't check if the quest has ended, and the hash doesn't contain any kind of deadline. A user may receive a signed hash and mint the receipt at any point in time. 

The quest owner can withdraw remaining tokens after the quest end time using the `withdrawRemainingTokens` present in the quests contracts. This is the implementation for `Erc20Quest`:

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81-L87

```solidity
function withdrawRemainingTokens(address to_) public override onlyOwner {
    super.withdrawRemainingTokens(to_);

    uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
    uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
    IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
}

function receiptRedeemers() public view returns (uint256) {
    return questFactoryContract.getNumberMinted(questId);
}
```

The function calculates how many receipts have been minted but are pending to be claimed, in order to leave the funds in the contract so the user can still claim those. However, this won't take into account receipts that are still pending to be minted.

## Impact

A user can mint the receipt for completing the task after the quest has ended, and in particular, if this is done after the owner of the quest has called `withdrawRemainingTokens`, then the user won't be able to claim the reward associated with that receipt.

This occurs because the user can mint the receipt after the quest end time, while the owner may have already withdrawn the remaining tokens, which only accounts for previously minted receipts.

Given this scenario, the user won't be able to claim the rewards, the contract won't have the required funds.

## PoC

In the following test, Alice mints her receipt after the quest owner has called `withdrawRemainingTokens`. Her call to `quest.claim()` will be reverted due to insufficient funds in the contract.

```solidity
contract AuditTest is Test {
    address deployer;
    uint256 signerPrivateKey;
    address signer;
    address royaltyRecipient;
    address minter;
    address protocolFeeRecipient;

    QuestFactory factory;
    ReceiptRenderer receiptRenderer;
    RabbitHoleReceipt receipt;
    TicketRenderer ticketRenderer;
    RabbitHoleTickets tickets;
    ERC20 token;

    function setUp() public {
        deployer = makeAddr("deployer");
        signerPrivateKey = 0x123;
        signer = vm.addr(signerPrivateKey);
        vm.label(signer, "signer");
        royaltyRecipient = makeAddr("royaltyRecipient");
        minter = makeAddr("minter");
        protocolFeeRecipient = makeAddr("protocolFeeRecipient");

        vm.startPrank(deployer);

        // Receipt
        receiptRenderer = new ReceiptRenderer();
        RabbitHoleReceipt receiptImpl = new RabbitHoleReceipt();
        receipt = RabbitHoleReceipt(
            address(new ERC1967Proxy(address(receiptImpl), ""))
        );
        receipt.initialize(
            address(receiptRenderer),
            royaltyRecipient,
            minter,
            0
        );

        // factory
        QuestFactory factoryImpl = new QuestFactory();
        factory = QuestFactory(
            address(new ERC1967Proxy(address(factoryImpl), ""))
        );
        factory.initialize(signer, address(receipt), protocolFeeRecipient);
        receipt.setMinterAddress(address(factory));

        // tickets
        ticketRenderer = new TicketRenderer();
        RabbitHoleTickets ticketsImpl = new RabbitHoleTickets();
        tickets = RabbitHoleTickets(
            address(new ERC1967Proxy(address(ticketsImpl), ""))
        );
        tickets.initialize(
            address(ticketRenderer),
            royaltyRecipient,
            minter,
            0
        );

        // ERC20 token
        token = new ERC20("Mock ERC20", "MERC20");
        factory.setRewardAllowlistAddress(address(token), true);

        vm.stopPrank();
    }

    function signReceipt(address account, string memory questId)
        internal
        view
        returns (bytes32 hash, bytes memory signature)
    {
        hash = keccak256(abi.encodePacked(account, questId));
        bytes32 message = ECDSA.toEthSignedMessageHash(hash);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(signerPrivateKey, message);
        signature = abi.encodePacked(r, s, v);
    }

    function test_Erc20Quest_UserCantClaimIfLateRedeem() public {
        address alice = makeAddr("alice");

        uint256 startTime = block.timestamp + 1 hours;
        uint256 endTime = startTime + 1 hours;
        uint256 totalParticipants = 1;
        uint256 rewardAmountOrTokenId = 1 ether;
        string memory questId = "a quest";

        // create, fund and start quest
        vm.startPrank(deployer);

        factory.setQuestFee(0);

        Erc20Quest quest = Erc20Quest(
            factory.createQuest(
                address(token),
                endTime,
                startTime,
                totalParticipants,
                rewardAmountOrTokenId,
                "erc20",
                questId
            )
        );

        uint256 rewards = totalParticipants * rewardAmountOrTokenId;
        deal(address(token), address(quest), rewards);
        quest.start();

        vm.stopPrank();

        // Alice has the signature to mint her receipt
        (bytes32 hash, bytes memory signature) = signReceipt(alice, questId);

        // simulate time elapses until the end of the quest
        vm.warp(endTime);

        vm.prank(deployer);
        quest.withdrawRemainingTokens(deployer);

        // Now Alice claims her receipt and tries to claim her reward
        vm.startPrank(alice);

        factory.mintReceipt(questId, hash, signature);

        // The following will fail since there are no more rewards in the contract
        vm.expectRevert();
        quest.claim();

        vm.stopPrank();
    }
}
```

## Recommendation

Since tasks are verified off-chain by the indexer, given the current architecture it is not possible to determine on-chain how many tasks have been completed. In this case the recommendation is to prevent the minting of the receipt after the quest end time. This can be done in the `mintReceipt` by checking the `endTime` property which would need to be added to the `Quest` struct or by including it as a deadline in the signed hash.
