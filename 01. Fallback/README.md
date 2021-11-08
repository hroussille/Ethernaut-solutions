# 01. Fallback

You will beat this level if

you claim ownership of the contract
you reduce its balance to 0.


## Target contract :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

In order to drain all funds we obviously need to call the withdraw function :

```solidity
  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }
```
But we must pass the onlyOwner modifier to do so...

```solidity
  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }
```

However, looking at the receive function :

```solidity
  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
```

We can see that sending funds to the contract address without data will result in a call to the receive function.
This function will give us ownership as long as we did a non zero contribution and msg.value is greater than 0.

To solve this challenge it is required to 

1. Send more than 0 ETH but less than  0.001 ETH to the contribute function.
2. Send any non 0 amount of ETH to the smart contract address, which will claim ownership.
3. Call the withdraw function. 
