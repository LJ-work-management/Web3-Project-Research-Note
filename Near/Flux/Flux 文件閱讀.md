https://docs.fluxprotocol.org/docs/

## 簡介
Flux 是一個跨鏈預言機聚合器，它為智能合約提供對任何事物的經濟安全數據源的訪問。該協議的每個組件都旨在最大限度地開源、無需許可和無需信任，使個人和社區能夠設計和開發定制的數據提供和檢索解決方案。

![](https://1460685071-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-M7DGgktMWPHxPSqnBds%2F-Mf8a4R2FQ2eTwjTKJnk%2F-Mf9FgrfGqH1K99DJy0n%2Fflux%20dark.svg?alt=media&token=b9fb15a0-804d-4449-b22b-bcbe3f849400)

Flux 由瑞士非營利實體 Open Oracle Association 支持和開發，其唯一目的是通過 USDC 和 Flux 代幣以貨幣方式分配贈款來支持生態系統。

### Oracle
Flux 提供兩種類型的 Oracle，第一方 Oracle (FPO) 和第三方 Oracle (TPO)。目前，Flux 已經在[NEAR](https://near.org)網絡上部署了 Oracle，在[Aurora](https://www.aurorachain.io)網絡和 NEAR 測試網上部署了 FPO。
如果希望在這些網絡上獲得價格對的 DeFi 協議，建議您從 FPO 開始，因為已經加入了 DeFi 中一些最高質量的數據提供商，並暫時補貼他們的費用！
此外，如果是數據提供商，有興趣在 FPO 上添加我們廣泛的價格信息，還可以啟動一個提供商節點以開始使用！
雖然已經在 NEAR 網絡上部署了 TPO 的第一個版本，但仍在開發 V2 的完全去中心化解決方案。

### First Party Oracle
FPO 提供了一個平台，數據提供者可以在該平台上使用開源的[Provider Node](https://github.com/fluxprotocol/oracle-provider-node)直接將其數據推送到鏈上，請求者可以**立即**查詢該數據。數據提供者使用他們自己的來源、聚合函數、硬件和軟件來收集數據，準備使用，並將其部署在鏈上。使用 FPO，數據饋送的可靠性、質量和供應掌握在提供商手中，免除了我們或任何其他方在途中操縱數據的能力。但是，數據僅與來源一樣可靠，如果出現問題，供應商的聲譽是唯一需要考慮的因素。如果您有興趣了解更多信息，請轉到[下一頁！](/docs/#first-party-oracle)

此外，如果您急於在 EVM 鏈上獲取數據並且等不及我們使用 FPO 合約和提供者節點設置我們的數據提供者，請在[此處](/docs/getting-started/first-party-oracle/fpo-providers)進行設置。

### Third Party Oracle
第三方預言機是完全去中心化、無許可、博弈論、經濟保證的預言機解決方案。雖然第 1 版已經上線，但我們目前正在開發我們的第 2 版，以使其更加健壯、公平，並將在今年對其進行測試和部署。
這個產品的目標是消除現有預言機解決方案帶來的中心化的各個方面，同時保持業內最高質量的數據供應。
我們的 TPO 將允許**任何人**通過使其輕量和廉價來啟動他們自己的 Provider Node。**您**決定您的來源、您的風險敞口（股權價值）、您的聚合/數據清理方法，以及您想要提供答案的請求。

使用我們專有的共識算法，您的數據響應將與其他驗證器的數據響應進行比較，直到自動達成共識。**每個**Validator 的錢都是在線的，所以不良行為者受到懲罰，誠實行為者得到獎勵！

我們的使命是消除每個網絡上每個協議的數據操縱攻擊的可能性；攻擊將變得比被盜的支出更昂貴。抓住這個社區的無許可、去中心化、自治的精神，控制權就在你手中！

目前，您可以將 NEAR 測試網上的 V1 TPO 用作**數據驗證器**或**數據請求器。**請求者創建智能合約來資助正在創建的特定於其需求的數據請求。驗證器為請求者提出的數據請求提供答案。驗證者手動輸入答案或運行節點來回答自動問題。
#### Data Validator
認為你有答案？運行驗證器節點或通過 Oracle Explorer 在 Flux Oracle 上手動解析數據
#### Data Requesters
有一個問題？需要安全的定價數據？部署我們的示例請求者合同並親自試用 Oracle！
#### Delegated Staking(即將推出)
無需任何技術知識來幫助為系統提供動力——只需將您的代幣與另一個驗證者進行質押！

https://docs.fluxprotocol.org/docs/getting-started/first-party-oracle

## First Party Oracle(FPO)


FPO 提供了一個平台，數據提供者可以在該平台上使用我們專有的提供者節點直接將他們的數據推送到鏈上。數據提供者使用他們自己的來源、聚合函數和硬件來收集他們的數據，準備使用，並將其部署在鏈上。使用 FPO，數據饋送的可靠性、質量和供應掌握在提供商手中，免除了我們或任何其他方在途中操縱數據的能力。數據與其來源一樣可靠，如果出現問題，供應商的聲譽也岌岌可危。
![[Pasted image 20220402103218.png]]
First Party Oracle Data Flow Chart

如何運作
1. 數據提供者已經有 API 端點/數據源用於獲取他們的數據
2. 他們將提供者節點部署到他們的服務器
3. 提供者節點定期調用端點並在端點內所需的源路徑收集數據
4. 如果數據檢索沒有錯誤，則提供者節點調用 FPO 合約並將數據發佈到鏈上的預期目的地 
5. 最終用戶/協議通過跨合約調用或 CLI 查詢合約，可以隨意使用結果數據

### Layers of Trustlessness (不信任層)
由於我們的工作方式與我們打算支持的協議一樣敏捷，因此我們為協議提供了不同的選項，以了解如何根據它們需要啟動的速度來獲取提供給它們的數據。
#### 您的資源，您部署
希望**立即**使用其獲取的任何數據引導自己的協議可以選擇部署自己的 FPO 智能合約和提供程序節點，以便它們處理數據源、聚合、清理和鏈上推送。我們提供工具，您負責其他一切。[如何使用](/docs/getting-started/first-party-oracle/fpo-providers#a-quick-brief)

### 我們的資源，我們部署
假使協議可以稍等片刻，並且有一些我們的高質量數據提供者無法提供的來源，可以選擇將他們的來源發送給我們，我們會將它們添加到我們自己的提供者節點並將他們的數據推送到鏈上。您仍然選擇來源，我們將處理鏈上數據的配置和推送。

### 提供者來源，證明者部署
對於希望以**最安全**的方式執行此操作的協議，這是最佳選擇。我們的高質量數據提供者設置他們的提供者節點來滿足您的數據饋送需求，並在推送到鏈上之前處理所有的清理和聚合。您的最終用戶可以高枕無憂，因為您無法控制價格，我們無法控制價格，並且數據來自一些最有信譽的 DeFi 來源。

以下介紹該如何使用
### FPO Requesters
https://docs.fluxprotocol.org/docs/getting-started/first-party-oracle/fpo-requesters
### FPO Providers
https://docs.fluxprotocol.org/docs/getting-started/first-party-oracle/fpo-providers

## Third Party Oracle


## [White paper](https://docs.fluxprotocol.org/docs/research-documentation/flux-oracle-v1-whitepaper)

https://docs.fluxprotocol.org/docs/research-documentation/flux-oracle-v1-whitepaper