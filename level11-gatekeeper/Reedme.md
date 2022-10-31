
The task is the following:

Make it past the gatekeeper and register as an entrant to pass this level.

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


I have started with the following concepts:

[Difference between tx.origin and msg.sender](https://ethereum.stackexchange.com/questions/1891/whats-the-difference-between-msg-sender-and-tx-origin)

[Gasleft function](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html#block-and-transaction-properties)

[Data types conversions](https://docs.soliditylang.org/en/v0.8.3/types.html#explicit-conversions)

In 2 words, the difference between **tx.origin** and **msg.sender**, is that 1st stores the address of a person started transaction (your wallet address) and the 2nd keeps the address of the previous initiator of the call (e.g. contract)
*Imagine you invoke Contract A - funcA() , which invokes Contract B funcB(). For contract A, both msg.sender and tx.origin would be the same(your address), but for contract B, msg.sender would be contract A, whereas tx.origin would be you.*

So to pass the **First gate** the only thing we need is intermediate contract between us and **GatekeeperOne** contract, we are trying to hack.

The second one is a bit triclky one so lets first go through the 3rd gate:
``Solidity
  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
```

Here is the conception we will need from the link above:
``` solidity
bytes2 a = 0x1234;
uint32 b = uint16(a); // b will be 0x00001234
uint32 c = uint32(bytes4(a)); // c will be 0x12340000
uint8 d = uint8(uint16(a)); // d will be 0x34
uint8 e = uint8(bytes1(a)); // e will be 0x12
```

simply saying, there are 2 things to highlight:
**uint16 = bytes2  (value in dex = uint / 4; bytes * 2 => uint16 = 0x0000; bytes2 = 0x0000)**
**uints are cutting\adding values left, bytes cutting\adding values right**

thus: 
  require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)) => **(bytes8 _gateKey)** will be the same HEX value as **uint64(_gateKey)**, which will be cut to **4 last digits in uint16**. **uint32(uint64(_gateKey)** is taking **last HEX 8 digits** from  **(bytes8 _gateKey) aka uint64(_gateKey) ** 
  
  require(uint32(uint64(_gateKey)) != uint64(_gateKey)
  require(uint32(uint64(_gateKey)) == uint16(tx.origin)

