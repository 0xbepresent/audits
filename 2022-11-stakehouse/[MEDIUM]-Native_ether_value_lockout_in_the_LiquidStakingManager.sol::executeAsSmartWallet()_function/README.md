# Original link
https://github.com/code-423n4/2022-11-stakehouse-findings/issues/174
# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L202


# Vulnerability details

## Impact

The ```LiquidStakingManager.sol::executeAsSmartWallet()``` function helps to interacts with a ```target``` contract (_to) as a ```SmartWallet``` contract. The problem is that the ```executeAsSmartWallet()``` function is payable but the received ETH msg.value is not controlled by the function and the msg.value is not transferred to the target contract. 

The purpose of the ```executeAsSmartWallet``` function is to interact with the target contract (_to) including the native ether.

## Proof of Concept

I wrote a test for the ```executeAsSmartWallet``` function:

1. Set up users, eth and network
2. Execute manager.executeAsSmartWallet with 2 ether attached and value parameter 0.
3. Manager balance will have 2 eth and the target contract will have zero. The 2 eth should be transferred to the target contract (MockTargetContract)

**Test:**

```solidity
// MockTargetContract.sol
// SPDX-License-Identifier: MIT

pragma solidity 0.8.13;


contract MockTargetContract {

    string public data;

    function greet(
        string memory _data
    ) external payable {
        data = _data;
    }
}

// LiquidDtakingManager.t.sol
// $ forge test -m "test_0xbepresent_executeassmartwallet_with_msgvalue" -vvv

function test_0xbepresent_executeassmartwallet_with_msgvalue() public {
    //
    // Native ether value lockout in the LiquidStakingManager.sol::executeAsSmartWallet() function
    // 1. Set up users, eth and network
    // 2. Execute manager.executeAsSmartWallet with 2 ether attached and value parameter 0.
    // 3. Manager balance will have 2 eth and the target contract will have zero. The 2 eth should be transferred to the target contract (MockTargetContract)
    //
    // 1. Set up users, eth and network
    //
    address nodeRunner = accountOne; vm.deal(nodeRunner, 4 ether);
    address feesAndMevUser = accountTwo; vm.deal(feesAndMevUser, 4 ether);
    address savETHUser = accountThree; vm.deal(savETHUser, 24 ether);
    depositStakeAndMintDerivativesForDefaultNetwork(
        nodeRunner,
        feesAndMevUser,
        savETHUser,
        blsPubKeyFour
    );
    //
    // 2. Execute manager.executeAsSmartWallet with 2 ether attached and value parameter 0.
    //
    MockTargetContract targetcontract = new MockTargetContract();
    vm.deal(admin, 10 ether);  // Give the admin 10 ether
    vm.startPrank(admin);
    manager.executeAsSmartWallet{value: 2 ether}(  // Send 2 ether to manager.executeAsSmartWallet()
        nodeRunner,
        address(targetcontract),
        abi.encodeWithSelector(
            MockTargetContract.greet.selector,
            "hello"
        ),
        0  // 0 value parameter
    );
    //
    // 3. Manager balance will have 2 eth and the target contract will have zero. The 2 eth should be transferred to the target contract (MockTargetContract)
    //
    assertEq(address(manager).balance, 2 ether);  // manager get 2 ether
    assertEq(admin.balance, 8 ether);  // admin dao remains 2 eth
    assertEq(address(targetcontract).balance, 0);  // The target is zero balance
    assertEq(targetcontract.data(), "hello");
    vm.stopPrank();
}
```

## Tools used
Foundry/VisualStudio

## Recommended Mitigation Steps

The function ```executeAsSmartWallet``` should check and send the msg.value to the target contract otherwise the msg.value will be stock in the LiquidStakingManager contract.

