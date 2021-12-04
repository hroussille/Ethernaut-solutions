# 17. Recovery

A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent 0.5 ether to obtain more tokens. They have since lost the contract address.

This level will be completed if you can recover (or remove) the 0.5 ether from the lost contract address.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

## Vulnerability

There is no real vulnerability here aside from the `destroy` function not being protected but that's a requirement to solve the challenge.
The core of the challenge is to find out the contract address.. Just use a block explorer to track down your initial tx to the contract holding the 0.5 ETH, it's very easy.

Once you have the address you need to call the `destroy` function to get the funds back. For example with Web3 :

```js
// Get the accounts from Metamask
const accounts = await web3.eth.getAccounts();
// ABI Encode the call to the destroy function forwarding funds to your personal address
const txData = web3.eth.abi.encodeFunctionCall({name: "destroy", type:'function', inputs:[{type:'address', name:'to'}]}, [accounts[0]]);
// Send your tx and solve the challenge
await web3.eth.sendTransaction({to: 'THE-CONTRACT-ADDRESS', from: accounts[0], data: txData});
```