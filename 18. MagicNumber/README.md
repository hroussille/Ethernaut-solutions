# 18. MagicNumber

To solve this level, you only need to provide the Ethernaut with a Solver, a contract that responds to whatIsTheMeaningOfLife() with the right number.

Easy right? Well... there's a catch.

The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 opcodes at most.

## Target Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

## Vulnerability

No vulnerability here, just a low level challenge where we need to build a tiny contract.

Using yul I built the following contract and compiled it with solc:

```yul
object "EmptyContract" {
  code {
    datacopy(0, dataoffset("Runtime"), datasize("Runtime"))
    return(0, datasize("Runtime"))
  }
  object "Runtime" {
    code {
      let answer := 42
      mstore(0x40, answer)
      return(0x40, 0x20)
    }
  }
}
```

This contract compiles to : `600a80600e600039806000f350fe602a60405260206040f3`.
Let's break this down into : `600a80600e600039806000f350fe` and `602a60405260206040f3`.

### Deployement bytecode

`600a80600e600039806000f350fe`

If we break that down into OPCODES:

- 0x60 : PUSH1
- 0x0a : 10
- 0x80 : DUP1
- 0x60 : PUSH1
- 0x0e : 14
- 0x60 : PUSH1
- 0x00 : 0
- 0x39 : CODECOPY
- 0x80 : DUP1
- 0x60 : PUSH1
- 0x00 : 0
- 0xf3 : RETURN
- 0x50 : POP
- 0xfe : INVALID

Which copies the current contract code from the 14th byte to the 24th : `602a60405260206040f3` to return it to the EVM, thus setting the runtime bytecode.

### Runtime bytecode

`602a60405260206040f3`

If we break that down into OPCODES:

- 0x60 : PUSH1
- 0x2a : 42
- 0x60 : PUSH1
- 0x40 : 64
- 0x52 : MSTORE
- 0x60 : PUSH1
- 0x20 : 32
- 0x60 : PUSH1
- 0x40 : 64
- 0xf3 : RETURN


Which pushes 42 (0x2a) to the stack, copies that value to memory at address 64 (0x40) (Free memory pointer).
Then Pushes 32 (0x20) to the stack : the size of our value which is on 32 bytes, alongside 64 (0x40) which is the address at which our value is stored to return the value we just copied to memory.


### Solution

```js
const accounts = await web3.eth.getAccounts();
await web3.eth.sendTransaction({to: '', from: accounts[0], data: "600a80600e600039806000f350fe602a60405260206040f3"});
await contract.setSolver("MY-DEPLOYED-CONTRACT-ADDRESS");
```

You can now submit the challenge and move to the next one !