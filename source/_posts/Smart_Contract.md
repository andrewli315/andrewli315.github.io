---
title: EIP 4337
date: {{ date }}
cover   : "/img/blockchain.jpg"
---
Smart Contract
===
偶然間找到之前的筆記，以此記錄一下


### Background

![](/img/20230319233934.png)


![](/img/20230319233942.png)

## EIP 4337
![EIP 4337](/img/20230319235339.png)
### UserOperation
> User 是 Signer 並不是 Account

使用者先建立一個以 `UserOperation` 這個 struct 的物件為基礎之合約呼叫
並且將其簽章之後傳遞給 `Bundler` ( 可以是一個合約或是 Relayer )

使用者需要先以 EIP 1014 計算出 UserOperation.sender的地址，也就是透過 `create2` 所創建的 Wallet Contract 地址 (Account Absatraction 合約地址)

### Bundler

https://github.com/eth-infinitism/bundler

Bundler 有可能是礦工，或是能作為 User 和 Miner 之間的中介人
> 這邊是透過合約去實作
> Bundler 其實應該算是EOA
> 自己運行的去中心化節點
> 可以視為 L2 Scale (Rollup )

會去聆聽 UserOperation mempool 中新加入的 `UserOperation`，並且將從這些交易中挑選一部分綑綁成 Bundle Transactions。

> _這邊我們可以假設 UserOperation 一定會被打包處理，因為可能會有類似 Flash Bot 的 Bundler 去快速過濾出有利潤（手續費）可圖的交易。_

首先 Bundler 會在 Local 使用 RPC Call 呼叫 Entry Point Smart Contract 中的 `simulateValidation()`，以此先確保這些 UserOperation 的簽章沒問題和 Gas Fee 被正常支付。關於模擬交易結果的部分可見

### Bundle Transaction

由 bundler 所打包的一個具有原子性 (atomic) 的合約操作(呼叫)
透過 bundler 送過去給 EntryPoint 驗證簽章，並且執行實際上的合約操作


### EntryPoint
EntryPoint 可以分成幾個部分
* Verification Phase
	* `_validatePrepayment` -   `_validateAccountPrepayment`
		* 若UserOperation沒有一個對應之 Wallet Contract 則會透過 initCode 這個 field 來建立一個新的合約
		* 若首次呼叫卻沒有提供此 initCode，則這個呼叫會失敗
		* Wallet Contract Address 與 UserOperation.sender 不同的話也會失敗
	* `_validateAccountAndPaymasterValidationData(uint256 opIndex, uint256 validationData, uint256 paymasterValidationData, address expectedAggregator)`
		* 檢查傳入的validationData 和 payMaster ValidationData
		* 
		* `_getValidationData(validationData)`
			* `_parseValidationData`
				* `return ValidationData(address, uint48, uint48)`
			* `aggregator = data.aggregator`
			* `outOfTimeRange = block.timestamp > data.validUntil || block.timestamp < data.validAfter;`
			* `return (aggregator, outOfTimeRange)
		* `expectedAggregator == aggregator`
		* `_getValidationData(paymasterValidationData)`
		* PayMaster 是一個幫忙付錢執行`UserOperation` 的帳戶 ( suppose 是一個wallet contract )
		* 
```
|    address of Aggregator    |  validUntil |  validAfter  |  
|             160 bits        |   48 bits   |   48 bits    |
|                          256 bits                        |

ValidationData{
	address aggregator;
	uint48 validAfter;
	uint48 validUntil;
}	     
```
* Execution Phase
	* 解析 UserOperation
	* 執行 UserOperation function call: `address(userOps.sender).call{value:0}(calldata)` 
	* 直接在 Wallet Contract 執行此操作 ( 所以 wallet contract 必須可以解析此function call 所夾帶之參數等等 )


#### Note
* Contract Wallet vs EOA ( External Owned Account ) 
	* EOA : 
		* 表示存款帳戶並且提供 address 讓人可以轉帳給此帳戶
		* 私鑰管理由其他機制提供管理 ex Custody, 軟硬體錢包
	* Wallet Contract :
		* 支付gas
		* 合約操作的功能

##### 以太坊帳戶設計
|           |   External Owned Account (EOA) 外部擁有帳戶 | Contract Account (CA) 合約帳戶 |
| ------ |  -------------------------------------------------- | -----------------------------------| 
| Property | 可以接收，持有或發送代幣，並與已部署的智慧合約進行互動。基於 Keccak與橢圓曲線(SECP256K1)公私鑰對。| 可以接受Ether，被EOA驅動時才可以進行其他操作。|
| Cost |  Free | Pay Gas to deploy a Contract |
| Transaction | 可以發起、執行任何交易，可以支付 gas，當執行交易時，第一個帳戶就是EOA，並且支付gas (不管後續contract 怎麼呼叫外部合約都是由第一個帳戶來支付) | 只能接受交易時發送交易(ex swap, transfer)，不能支付gas||
|  Operation | EOA 之間只能進行代幣transfer ( 原生代幣 ETH transfer )，不能是其他交易 | EOA 向合約帳戶發起交易可已執行多種操作，例如轉移代幣或是部署新的合約 (合約如何寫，就可以如何操作)|
| Control |  balance : 餘額，nonce : 歷史交易次數，address : 私鑰的64bytes, 公鑰: 0x+後20bytes公鑰, codeHash: Null, storageRoot | balance: 餘額， nonce : 創建了的合約數量，address: 43bytes，codeHash: 帳戶程式對應之hash值。 storageHash: Merkle Patricia trie Root的Hash值|


#### 帳戶抽象化
* 將EOA 跟 CA 所提供的功能，整合成一個AA(Account Abstraction)
* EIP-86 - 2017
* EIP 2938 - 2020
* EIP 4337 - 2021



#### 議題
帳戶抽象化 : EOA 存在的議題
* 無論是託管還是使用者自行管理，都會造成私鑰管理與註記詞上的問題
* 讓使用者保存這些東西可能就是一個可以待解決的問題
* 託管的話就要考量access control 與託管機構是否真的有效且適當的儲存

