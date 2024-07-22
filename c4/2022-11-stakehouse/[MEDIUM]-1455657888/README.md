# Original link
https://github.com/code-423n4/2022-11-stakehouse-findings/issues/377
# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L202-L215
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/smart-wallet/OwnableSmartWallet.sol#L52-L64


# Vulnerability details

## Impact
Calling the `executeAsSmartWallet` function by the DAO further calls the `OwnableSmartWallet.execute` function. Since the `executeAsSmartWallet` function is `payable`, an ETH amount can be sent when calling it. However, since the sent ETH amount is not forwarded to the smart wallet contract, such sent amount can become locked in the `LiquidStakingManager` contract. For example, when the DAO attempts to call the `executeAsSmartWallet` function for sending some ETH to the smart wallet so the smart wallet can use it when calling its `execute` function, if the smart wallet's ETH balance is also higher than this sent ETH amount, calling the `executeAsSmartWallet` function would not revert, and the sent ETH amount is locked in the `LiquidStakingManager` contract while such amount is deducted from the smart wallet's ETH balance for being sent to the target address. Besides that this is against the intention of the DAO, the DAO loses the sent ETH amount that becomes locked in the `LiquidStakingManager` contract, and the node runner loses the amount that is unexpectedly deducted from the corresponding smart wallet's ETH balance.

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L202-L215
```solidity
    function executeAsSmartWallet(
        address _nodeRunner,
        address _to,
        bytes calldata _data,
        uint256 _value
    ) external payable onlyDAO {
        address smartWallet = smartWalletOfNodeRunner[_nodeRunner];
        require(smartWallet != address(0), "No wallet found");
        IOwnableSmartWallet(smartWallet).execute(
            _to,
            _data,
            _value
        );
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/smart-wallet/OwnableSmartWallet.sol#L52-L64
```solidity
    function execute(
        address target,
        bytes memory callData,
        uint256 value
    )
        external
        override
        payable
        onlyOwner // F: [OSW-6A]
        returns (bytes memory)
    {
        return target.functionCallWithValue(callData, value); // F: [OSW-6]
    }
```

## Proof of Concept
Please add the following code in `test\foundry\LSDNFactory.t.sol`.

1. Add the following `receive` function for the POC purpose.
```solidity
    receive() external payable {}
```

2. Add the following test. This test will pass to demonstrate the described scenario.
```solidity
    function testETHSentWhenCallingExecuteAsSmartWalletFunctionCanBeLost() public {
        vm.prank(address(factory));
        manager.updateDAOAddress(admin);

        uint256 nodeStakeAmount = 4 ether;
        address nodeRunner = accountOne;
        vm.deal(nodeRunner, nodeStakeAmount);

        address eoaRepresentative = accountTwo;

        vm.prank(nodeRunner);
        manager.registerBLSPublicKeys{value: nodeStakeAmount}(
            getBytesArrayFromBytes(blsPubKeyOne),
            getBytesArrayFromBytes(blsPubKeyOne),
            eoaRepresentative
        );

        // Before the executeAsSmartWallet function is called, the manager contract owns 0 ETH,
        //   and nodeRunner's smart wallet owns 4 ETH. 
        assertEq(address(manager).balance, 0);
        assertEq(manager.smartWalletOfNodeRunner(nodeRunner).balance, 4 ether);

        uint256 amount = 1.5 ether;

        vm.deal(admin, amount);

        vm.startPrank(admin);

        // admin, who is dao at this moment, calls the executeAsSmartWallet function while sending 1.5 ETH
        manager.executeAsSmartWallet{value: amount}(nodeRunner, address(this), bytes(""), amount);

        vm.stopPrank();

        // Although admin attempts to send the 1.5 ETH through calling the executeAsSmartWallet function,
        //   the sent 1.5 ETH was not transferred to nodeRunner's smart wallet but is locked in the manager contract instead.
        assertEq(address(manager).balance, amount);

        // Because nodeRunner's smart wallet owns more than 1.5 ETH, 1.5 ETH of this smart wallet's ETH balance is actually sent to address(this).
        assertEq(manager.smartWalletOfNodeRunner(nodeRunner).balance, 4 ether - amount);
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L210-L214 can be updated to the following code.

```solidity
        IOwnableSmartWallet(smartWallet).execute{value: msg.value}(
            _to,
            _data,
            _value
        );
```