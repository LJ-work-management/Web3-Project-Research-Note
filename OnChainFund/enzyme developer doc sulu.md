# Enzyme Sulu Developer Doc
###### tags: `Enzyme doc read`
doc: https://specs.enzyme.finance/architecture/persiste

Contracts：
用於生產和測試的所有合同都位於`contracts/`

`mocks/`- 僅用於測試的模擬合約
`persistent/`- 在首次部署後不會更改的生產合同。這些合約可用於多個版本，例如，促進資金從舊版本遷移到當前版本的總體結構。
`release/`- 生命週期不超過此版本的生產合同。
`test/`- 僅用於測試的其他合約

## Architecture(架構)
### Persistent(永久架構)
Enzyme 具有可升級的基金，其結構為經理和投資者提供選擇加入（經理）或選擇退出（投資者）重大更新的機會。
基本狀態存在於 `VaultProxy`中，它通過升級`VaultLib`其版本在版本之間移動，並通過全局合同`Dispatcher` 獲得許可。
-   holdings
-   shares
-   roles

#### VaultProxy(金庫代理)
上面描述的“基本狀態”存在於 per-fund `VaultProxy`合約實例中，它們是遵循 EIP-1822 代理模式的可升級合約。
`VaultProxy` 指定  `VaultLib`作為目標邏輯，這些邏輯合約在每個版本中部署，並在遷移期間換出。
合約為每個`VaultLibBaseCore`  `VaultProxy` 定義了絕對必要的狀態和邏輯。這包括：
-   一個標準的 ERC20 實現，稱為`SharesTokenBase`
-   `IProxiableVault`的功能被`Dispatcher`呼叫
-   核心訪問控制角色：`owner`、`accessor`、`creator`和`migrator`
    
`owner`是基金的所有者。
`migrator`是一個可選角色，允許非所有者遷移基金。
`creator`是合約`Dispatcher`允許更新`accessor`和`vaultLib`值。
`accessor`是可以對`VaultProxy`進行狀態變更的基礎帳戶(這是發布等級的合約可以和基金內的資產做交互,更新balance)
`VaultProxy`是非常抽象的合約,能夠提供完美的更新和新增功能

#### Dispatcher(調度者)
Dispatcher是一個不可升級合約,用於
- 部署新版本的`VaultProxy`
- 遷移`VaultProxy`
- 保持全局狀態(擁有者(i.e., the Enzyme Council) and the default `symbol` value for fund shares tokens)

`Dispatcher` 儲存 `currentFundDeployer` (最新版本部署者),而且只有這個`msg.sender`可以呼叫功能來遷移`VaultProxy`
`FundDeployer` 可以選擇落實遷移由`IMigrationHookHandler`所提供的鉤子,讓版本能夠在`Dispatcher`在逕行前移時掉用鉤子時實現任意邏輯
`VaultLibBaseCore`使用`FundDeployer`這個抽象的概念,只關注(接觸optional callbacks的)身份

因此，從這些持久性合約的角度來看，發布級合約大多是任意的，為未來的迭代和更改提供了最大的靈活性。

---

#### Other
除了在所有版本中持續存在的核心持久架構之外，還有一些持久性組件的生命週期可以跨越一個或多個版本。
##### Protocol Fee Reserve
`ProtocolFeeReserveProxy`用作存儲收取的費用的存儲庫，具有可升級的邏輯，用於處理收集的資產。

##### External Position Factory
部署新的`ExternalPositionProxy`實例，這些實例是可升級的代理合約，用於管理位於保險庫外的頭寸，例如 CDPs。

CDPs?

---

### Release(發行版本架構)
#### Core(核心)
##### FundDeployer(基金部署者)
`FundDeployer`是發布版的起點,有兩個功能：
- 創建、遷移和重新配置資金的門戶
- 這是被`Dispatcher`視為的合約`currentFundDeployer`，因此允許它部署和遷移`VaultProxy`實例
- 發佈內遷移（通過更換新的 ComptrollerProxy 來完全更改基金配置，稱為“重新配置”），模仿`Dispatcher`'s 的範例來發出信號並執行時間鎖定的“重新配置請求”
- `FundDeployer` 部署配置合約(ComptrollerProxy), 然後將其附加到`VaultProxy`實例
- `FundDeployer`還用作發布範圍的註冊表，以對所有資金以統一的方式限制許可調用的允許值。
`
##### ComptrollerProxy(主計長代理)
`ComptrollerProxy`是按基金部署的(一個基金對應到一個)，它是此版本中與基金交互的規範合約。它存儲核心發布級別的配置透過`accessor`附屬在`VaultProxy`中
對`VaultProxy`中所有與基金持股和股票餘額相關的狀態更改調用都必須通過`ComptrollerProxy`，這使其成為訪問控制的一個至關重要的瓶頸。
`ComptrollerProxy`中存儲和邏輯在`ComptrollerLib`和其關聯的庫中定義,這是不可升級的,但可以透過`FundDeployer`部署新的`ComptrollerProxy`(重新配置)
##### VaultLib(金庫方法庫)
`VaultLib`合約包含附加到此版本`VaultProxy`實例的存儲佈局、事件簽名和邏輯
#### Extensions(擴展)
通過添加額外種類的功能來擴展核心合約的邏輯
它們是半信任的，因為它們被選擇性地授予對`VaultProxy`實例的狀態更改調用的訪問權限。
為了進行這樣的狀態改變調用，必須滿足兩個條件：
1. 擴展函數必須由`ComptrollerProxy`透過含有`allowsPermissionedVaultAction`修飾器的方法來調用
2. 改變狀態的調用必須通過`ComptrollerProxy`,並委託給`PermissionedVaultActionLib`來確定是否允許調用的 Extension 執行這樣的操作。
這樣的規範確立只有在`VaultProxy`對應到的`ComptrollerProxy`調用的擴展(還必須要被允許)才能改變`VaultProxy`的狀態
每個擴展管理不同的插件：
-   `IntegrationManager` - adapters
-   `PolicyManager` - policies
-   `FeeManager` - fees
-   `ExternalPositionManager` - external positions
插件無權對 Vault 的狀態進行操作，除非通過核心系統明確授予。例如，一個`SynthetixAdapter`可以代表一個保險庫在 Synthetix 上進行交易，方法是通過一個`ComptrollerLib.vaultCallOnContract()`
在此版本中，有四個擴展。所有基金每次延期共享一份合約。
##### PolicyManager(政策管理者)
這`PolicyManager`允許基金所有者設置和管理一組“政策”，這些“政策”用於在各種函數調用期間執行定制驗證。
該協議在認為重要的行動期間調用“政策掛鉤”，以根據他們的特定需求為基金所有者提供應該允許/禁止什麼的可定制性。
策略定義了它們實現了哪些鉤子。到達掛鉤時，它會遍歷基金已啟用的在該特定掛鉤上運行的所有策略，並驗證是否允許特定調用。
查看`IPolicyManager.PolicyHook`可用的掛鉤。
策略本身只能失敗或通過，因此`PolicyManager`不需要或訪問狀態更改的保險庫操作。
所有策略都可以在基金創建、遷移或重新配置期間添加。
策略自行定義它是否可更新或可移除。
可以隨時添加政策，除非它運行在限制當前投資者的政策掛鉤上（即股票贖回和股票轉讓）。

##### FeeManager(費用管理者)
根據`FeeManager`其內部邏輯，允許“費用”決定基金份額的鑄造、銷毀或轉讓。
與政策一樣，費用實施“費用掛鉤”，在特定操作期間調用。
費用可以立即結算和支付，也可以在結算時作為“流通股”累積，並且只有在滿足特定條件（由費用定義）時才解鎖支付。
費用只能在基金設置（創建/遷移/重新配置）期間添加或刪除。
##### IntegrationManager(整合管理者)
允許：`IntegrationManager`
-   通過 DeFi 協議的“adapters”（例如 Uniswap、Kyber、Compound、Chai）將基金資產交換為其他資產
-   追踪資產
-   取消追踪的資產
這些操作中的每一個都包含一個策略掛鉤。
##### ExternalPositionManager(擴展部位管理者)
這`ExternalPositionManager`允許創建和管理“外部頭寸”，代表該基金的非 ERC20 持股的代理合約，例如 Compound CDP。

`ExternalPositionManager`為每個外部位置維護一個“libs”和“parsers”的註冊表，在此版本中用作`ExternalPositionProxy`特定的beacon代理模式的"beacon"。

與外部頭寸交互的每個可用操作都包含一個策略掛鉤。

#### Plugins(插件)
每個擴展都使用插件。`IntegrationManager`使用“adapters”，`PolicyManager`使用“policies”，`FeeManager`使用“fees”，`ExternalPositionManager`使用“external positions”。
費用、政策和集成允許使用任意（第三方）插件。基金經理可以決定是否使用第三方插件，投資者將能夠確定基金配置對於他們的信任門檻是否安全。僅使用官方插件的設置需要：
-   沒有限制投資者行為的任意政策（只能在基金設立時添加）
-   沒有任意費用（只能在基金設置時添加）
-   沒有任意適配器（不可移除的策略將提供官方的、理事會驗證的適配器註冊表）
#### Infrastructure(基礎設施)
除了“Core”和“Extension”發布級合約之外，還有一大類“基礎設施”合約，它們是發布級協議的雜項依賴項。與擴展不同，它們沒有獲得任何更改基金狀態的權限。
##### AssetFinalityResolver(資產結算者)
`AssetFinalityResolver`依賴於準確的 Synth 餘額的操作， 以一種有效的方式結算Synths。
##### Gas Relayer(手續費中繼)
此版本允許資金有選擇地使用加油站網絡中繼器來支付對特定合同和功能的調用。
##### ProtocolFeeTracker(合約費用追蹤器)
`ProtocolFeeTracker`跟踪協議的費用支付，並被用以確定基金應向`ProtocolFeeReserve`鑄造的股份數量，以使其費用保持更新。
##### ValueInterpreter(值解釋器)
這`ValueInterpreter`是各種“price feeds”（僅由酶委員會管理的一種附加類型的“plugin”）的聚合器，用於根據輸出資產計算一個或多個輸入資產數量的價值.

此版本中有兩類資產：
- primitives(基礎資產):通過 Chainlink 聚合器獲得直接匯率的資產，可用於將一種基礎資產轉換為任何其他基礎資產（例如，WETH、MLN 等）
- derivatives(衍生資產):僅根據標的資產（例如 Chai、Compound cTokens、Uniswap 池代幣等）對其有利率的資產
`ValueInterpreter`決定資產是基礎資產還是衍生資產，並執行邏輯以使用相應的價格饋送來確定輸出資產中的價值
#### Interfaces(接口)
外部合約的所有接口都包含在`release/interfaces/`目錄中。
內部合約的接口（例如`IFundDeployer`）保存在它們所引用的合約旁邊。這些是狹窄的接口，只包含其他非插件、發布級合約（即上面的“Core”和“Extension”部分中的那些）所需的功能。

## Users
### End users
用戶有兩類：基金經理和投資者。
#### Fund managers
此版本中有三個主要的基金管理角色，存儲在`VaultProxy`：
-   Owner -> 
	- 可以對基金執行任何管理操作, 所有權可以通過提名交易來改變
-   Migrator ->
	-  每個基金可以有一個遷移者,可以在`FundDeployer`上調用遷移和重新配置
	- 只有owner可以設置和解除遷移者
-   Asset Manager -> 
	- 每個基金可以有許多資產經理, 可以任意調用`IntegrationManager`和`ExternalPositionManager`也可以回購協議費用份額,
	- 可以透過policies,縮小每個資產經理允許調用`IntegrationManager`和`ExternalPositionManager`的範圍
	- 只有owner可以設置和解除資產經理
### Administrators
在 Enzyme Protocol 中管理特權功能的主要有兩個方面：Avantgarde Core（部署者）和 Enzyme Council（管理員）。
##### Avantgarde Core（部署者）
作為協議的主要開發者，Avantgarde Core 部署所有合約，並在發布之前配置所有合約。一旦發布上線，受保護功能的完全訪問控制權將移交給酶委員會。
##### Enzyme Council DAO (管理者)
Enzyme Council DAO 由兩個小組委員會組成；Enzyme Technical Council (ETC) 和Enzyme Exposed Businesses (EEB)。ETC 是技術熟練的指定方的機構，當發布上線，指定擁有對保護功能和協議公能擁有唯一投票權的機構。

Enzyme Council是一個完全受信任的實體，它是協議安全假設的核心

### Access Control Handoff
#### Dispatcher ownership
`Dispatcher`的所有者是協議的全局管理員，跨版本持續存在（版本如何實現該權限取決於當下版本）。
Enzyme Council 能夠通過提名申請程序轉移所有權（例如，轉移到一個新的多重簽名錢包，就像已經發生的那樣）。
#### FundDeployer ownership
對於此版本，`FundDeployer`的所有者視為版本級協議合約的管理員。
`FundDeployer`的所有者是動態設置的：
#### Extensions and plugins ownership
需要所有權以進行訪問控制的擴展（`FeeManager`、`PolicyManager`、`IntegrationManager` ）和插件（fees, policies, integration adapters）將所有權給到當前所有者`FundDeployer`。這是通過從`FundDeployerOwnerMixin`繼承合約來實現的。
因此，當 的所有者`FundDeployer`成為 ETC 時，所有實現該 mixin 的合約的所有者也是如此。
這些移交訪問控制模式為部署和配置提供了最大的靈活性，同時確保一旦協議生效，ETC 最終將擁有完整的管理員權限。
## Topic
### Fund Lifecycle
在全局範圍內，可以根據`Dispatcher`. 把資金可以創建在最新版本。
創建後,基金可以透過遷移換到最新版本
這些操作的全局邏輯位於`Dispatcher`中，而附加的發布級邏輯位於`FundDeployer`.

`FundDeployer`作為發布合約的最上層級,管理：
-   **創建**：如何在此版本中創建新基金
-   **遷移**：以前版本的基金如何升級到此版本
-   **重新配置**：此版本的基金如何更改所有版本級別的配置（核心設置、政策和費用）
透過上述動作,一個新的`ComptrollerProxy`將被部署,並附加到`VaultProxy`當成其`accessor`
它存儲核心配置選項，並且與擴展及其插件相關的配置通過引用`ComptrollerProxy`. 
因此`ComptrollerProxy`，為基金的主要配置對象。
#### 創建
為了創建新基金，CallerA（任何賬戶）可以調用`FundDeployer.createNewFund()`，基金的所有組件和配置都是原子創建的。此功能採取的步驟是：
-  `FundDeployer`部署一個新`ComptrollerProxy`實例，該實例設置調用者提供的核心和擴展配置 
-  `FundDeployer`調用`Dispatcher`以使用 CallerA-provided`fundOwner`和`fundName`部署一個新的`VaultProxy`合約（請注意，`fundOwner`不需要是 CallerA），以及發布的`VaultLib`和新創建的`ComptrollerProxy`將成為`VaultProxy`的`accessor` .
-  `FundDeployer`在`ComptrollerProxy`上設置新部署的`VaultProxy`,並且調用`ComptrollerProxy.activate()`以執行最終設置邏輯，並為擴展提供最後一次驗證和更新狀態的機會。 
-  該基金現已在此版本中上線。
#### 遷移
##### 從以前的版本遷移到此版本
-  MigratorA 調用`FundDeployer.createMigrationRequest()`，部署一個新`ComptrollerProxy`實例，設置調用者提供的核心和擴展配置和將要被遷移的`VaultProxy`。
-  MigratorA 等待定義在`Dispatcher`上的`migrationTimelock`通過。  
-  MigratorA 調用`FundDeployer.executeMigration()`，`FundDeployer.executeMigration()`調用`Dispatcher.executeMigration()`
-   `Dispatcher`更新`VaultProxy`的`VaultLib`並將新創建的`ComptrollerProxy`分配成為其`accessor`.
-  `FundDeployer`調用`ComptrollerProxy.activate()`為擴展提供驗證和更新狀態的最後機會
-  該基金現已在此版本中上線。

##### 從此版本遷移到新版本
`Dispatcher`在遷移的每個階段(調用 outbound `FundDeployer`)觸發鉤子，使此版本有機會執行任意代碼，對遷移做出反應。
此版本僅實現`invokeMigrationOutHook`(運行在執行遷移之前)：
-   支付到期的協議費用
-   支付任何已發行的費用股
-   調用`ComptrollerProxy`上的`selfdestruct()`

##### 重新配置
重新配置與遷移（低級別和高級別）非常相似，因為在同一版本`ComptrollerProxy`中創建了一個新的來替換舊的。`ComptrollerProxy`它甚至可以稱為“版本內遷移”。
不同之處主要在於，與其將調用傳遞給權威`Dispatcher`，不如使用`FundDeployer`本身存儲`reconfigurationRequest`並定義一個本地的 `reconfigurationTimelock`。
1. MigratorA 調用`FundDeployer.createReconfigurationRequest()`，部署一個新`ComptrollerProxy`實例，設置調用者提供的核心和擴展配置，然後`VaultProxy`將被移動。
2. MigratorA 等待`reconfigurationTimelock`通過。
3. MigratorA 調用`FundDeployer.executeReconfiguration()`，它將新創建的`ComptrollerProxy`分配為其`accessor`.
4. 如“從此版本遷移到新版本”中所述，`FundDeployer`停用舊版本的`ComptrollerProxy`。
5. `FundDeployer`調用`ComptrollerProxy.activate()`為擴展提供驗證和更新狀態的最後機會
6. 該基金現在此版本中具有新的配置的`ComptrollerProxy`。
### Holdings and Shares
基金的核心功能是：
-   接受投資者**存款**
-   使用這些存款建立鏈上資產**組合**
-   促進**贖回**股票以獲得投資組合
每個基金都配置有“面額資產(作為對標的資產)”，這是計算 GAV和股價的記賬單位。
#### Holdings
GAV 中包含兩種類型的持股：“跟踪資產(Tracked Assets)”和“外部頭寸(external positions)”。
##### Tracked Assets
“跟踪資產”是可替代的、符合 ERC20 標準的資產，屬於`VaultProxy`，其狀態存儲為`trackedAssets`.
例如，WETH、MLN、Compound cTokens、Uniswap V2 LP 代幣或構成資產領域的任何其他資產。

每當協議識別出新資產已轉移到`VaultProxy`時,那個資產被列入跟踪資產,例如透過在`IntegrationManager`適配器中透過Defi項目所做的交易或是從外部頭寸提領資產回到`VaultProxy`
資產經理也可以通過專門`IntegrationManager`的操作（每個操作都有自己的策略掛鉤）明確添加或刪除跟踪資產，儘管基金的面額資產始終是跟踪資產。

在 Enzyme Protocol v3，基金持有的資產僅包含跟踪資產。
##### External Positions
從 Enzyme Protocol v4 開始，“外部頭寸”可作為第二種持有類型，用於操作不會導致簡單的 ERC20 資產交換的情況。
例如，Compound CDP、Uniswap v3 LP 倉位
外部頭寸`VaultProxy`作為不同的`ExternalPositionProxy`實例存在於不符合 ERC20 且不可分割的實例之外。這些頭寸本身可以持有有價值的資產（例如，作為 CDP 抵押品的 Compound cToken），或者僅代表在協議之外持有價值的頭寸的所有權，例如 Uniswap v3 LP 頭寸或質押資產。

##### 不包括：未追踪的資產和外部獎勵
值得注意的是，某些“屬於”基金的資產不包括在其 GAV 和股價中。
- “未跟踪資產”是屬於`VaultProxy`但不屬於“已跟踪資產”狀態的 ERC20 資產。
- “外部獎勵”是在外部協議中為藉貸、質押或以其他方式參與而產生的無人認領的資產，例如`COMP`在 Compound 上借貸而產生的無人認領的資產。
##### Shares
股票是可替代的 ERC20 代幣，代表了按總股票供應量的比例為持股提供資金的權利。
任何數量股份的標準價值是總基金 (GAV.mul(股份比例).div(總供應量))。

###### Deposits(存款)

存款方法一(直接調用)
存款時，用戶調用`ComptrollerProxy.buyShares()`一定數量的面額資產進行存款，`ComptrollerProxy`將面額資產金額轉移到`VaultProxy`，並在其中吸收到持有量中。然後將相對於當前股價的股票鑄造給存款人。

存款方法二(外圍合約)
access-controlled `ComptrollerProxy.buySharesOnBehalf()`由外圍合約(peripheral contracts, 即`DepositWrapper`）在存款期間包裝最終用戶的操作，例如從 AssetA 交易到面額資產然後存款。
重要的是要小心地控制代表他人存款的訪問權限，以免暴露由於`sharesActionTimelock`（參見下面的“轉賬”）而造成的惡意攻擊。

有一個策略掛鉤(`PolicyHook.PostBuyShares`) ,和兩個費用掛鉤 ( `FeeHook.PreBuyShares`,`FeeHook.PostBuyShares`) 在這兩個函數共享的公共`__buyShares()`邏輯期間運行，允許策略在更改基金持有量之前和之後立即驗證買家和投資金額和費用並分享供應。

###### Redemptions(贖回)
有兩種可用的贖回機制。

在這兩種情況下，股票都被燒毀以換取獲得相應的基礎資產。
在這兩種情況下，費用都可以在贖回之前通過`FeeHook.PreRedeemShares`.

`redeemSharesInKind()` 	直接拿回所有份額資產

相對的股份數量被贖回，指定的`_recipient`會收在`VaultProxy`到 ERC20 資產中的一部分。
默認情況下，這些資產僅限於 的“跟踪資產” `VaultProxy`，但贖回者可以指定忽略（即沒收）跟踪資產,或包含未跟踪資產（即，屬於`VaultProxy`但不是“跟踪資產”的 ERC20 代幣”）。
例如，FundA 發行了 10 個股票單位並持有 20 個 WETH 和 10 MLN。UserA 贖回 1 個份額（佔總供應量的 10%）。UserA 收到 2 WETH 和 1 MLN。

沒有政策可以在此功能上運行，因為它應該作為保證贖回選項持續可用，儘管在某些情況下無法贖回全部股票價值：
1. 基金持有的資產不可轉讓（例如，由於 ERC20 資產本身暫停，由於 Synth 餘額在 Synthetix 交易後尚未結算等）
2. 該基金持有“外部頭寸”的價值，這些頭寸不可分割，ERC20 representations，不包括在內

後一點很關鍵：除了極少數例外，被允許使用外部頭寸（由政策強制執行）的基金用戶不應以實物贖回股票，因為他們只會收到 ERC20 資產的一部分`VaultProxy`，並沒收在外部頭寸中持有的部位。

`redeemSharesForSpecificAssets()` 拿回指定一個或多個產(外部頭寸的基金的典型贖回方法)

贖回者指定一個或多個`VaultProxy`ERC20 持有量以及每個持有的相對價值（總計 100%）。

例如，FundA 以 DAI 計價，發行了 10 股單位，總 GAV 為 1000 DAI。UserA 贖回 1 個股份單位（總供應量的 10%，價值 100 DAI）並指定接收 75% 的 DAI 和 25% 的 MLN。UserA 收到 75 DAI 和價值 25 DAI 的 MLN。

例如，政策可以實施`PolicyHook.RedeemSharesForSpecificAssets` 以定義可以贖回的資產的限制。
與`redeemSharesInKind()`不同的是，此選項支付贖回`_recipient`其所欠價值的比例，包括存儲在外部頭寸中的價值。因此，該函數應用作任何使用外部頭寸的基金的典型贖回方法。
###### Shares Action Timelock
每個基金都配置自己的`sharesActionTimelock`，它定義了在用戶A 最後一次通過存款收到股票之後必須經過的秒數，然後才能被允許贖回或轉讓任何股票。

這是一種套利保護，有不受信任的投資人(放錢進入基金者(可能會惡意提領走某些特定資產?))的基金應該使用非零值。
###### Transfers(轉讓)
股票符合 ERC20 標準，默認情況下可以轉讓，但有一些驗證可以阻止轉讓：
1. 在 UserA 的“shares action timelock”到期之前，UserA 不能將股份轉讓給任何用戶
2. `PolicyHook.PreTransferShares`使策略能夠驗證傳輸條件（例如，允許接收者的白名單）

	因為基金配置（包括政策）可以通過遷移或重新配置來改變，所以第二點對於二級市場或任何其他智能合約持有人來說尤其成問題,例如：基金添加阻止從 Uniswap 轉移的政策池->LP 提供者將被困在不可撤回的位置。

如果沒有核心保證，即一旦他們的合約通過轉入獲得股份，他們將始終能夠將其轉出，那麼對於流動性提供者或建設者來說，這將進一步不切實際。

對於那些願意為二級應用程序的構建者或用戶提供此類保證的資金，因此有著永久的設置：(在`VaultProxy`上設置`freelyTransferableShares`). 此配置選項不會`PolicyHook.PreTransferShares`在基金轉移時運行，並且會在遷移和重新配置之間持續存在。一旦設置，就不能取消設置。


### External Positions(外部頭寸)
“外部頭寸”是 v4 中引入的一種基金持有類型，它：

-   不是可替代的 ERC20 代幣
-   作為不同的合約存在於保險庫之外（每個外部頭寸實例一個）
-   根據其相關持股和負債向基金回傳其價值
-   在股票贖回期間不能收回價值（使用外部頭寸的基金投資者幾乎不應該贖回實物，因為外部頭寸內部的任何價值都將被沒收）
-   是一個“信標”代理合約，使用基金當前版本規定的庫（例如，Sulu）
#### Persistent architecture 
`ExternalPositionProxy`由一個持久的`ExternalPositionFactory`部署，它作為外部位置“類型”（例如，Compound CDP）的註冊,和所有有效`ExternalPositionProxy`實例（即，通過工廠創建的實例)。

`ExternalPositionProxy`和`VaultProxy`一起被部署,`ExternalPositionProxy`作為`VaultProxy`的擁有者。

每次調用時，`ExternalPositionProxy`通過`getExternalPositionType()`回調其`VaultProxy`.

受保護/受信任的對狀態更改操作的調用`ExternalPositionProxy`應該通過受保護的`receiveCallFromVault()`，保證調用通過其擁有的`VaultProxy`。

#### Release architecture
外部頭寸的生命週期通過在`ExternalPositionManager`上的`ExternalPositionManagerActions`管理：
-   `CreateExternalPosition`-- 創建（部署）一個新的外部部位並在`VaultProxy`上啟動它
-   `CallOnExternalPosition`-- 允許任意調用與外部頭寸交互，以及對`VaultProxy`  進行的資產轉移
-   `RemoveExternalPosition`-- 從`VaultProxy`上停用外部頭寸
-   `ReactivateExternalPosition`-- 重新激活`VaultProxy`上現有的外部頭寸
這些操作中的每一個都有自己的策略掛鉤，以允許對風險管理進行精細控制。
每個`ExternalPositionManager`外部頭寸類型存儲兩個可更新的參考合約，其中包含特定於該類型的所有邏輯：
-   `lib`是信標`ExternalPositionProxy`的目標庫，包含用於本地記帳和與外部協議交互的業務邏輯
-   `parser`是被`ExternalPositionManager`使用的合約，用於驗證和解析對其外部頭寸類型的每個特定操作的調用數據
    

範例：
VaultA 有一個 Compound CDP 外部部位。要向 CDP 添加抵押品：
-   資產經理通過所需的有效負載調用`ExternalPositionManager`的`CallOnExternalPosition`操作,透過`ComptrollerProxy.callOnExtension()`添加 100 cDAI 作為抵押品
-   `ExternalPositionManager`查找其存儲的`CompoundDebtPositionParser`並將有效負載轉發給它
-   `CompoundDebtPositionParser`驗證目前的調用，例如cDAI 是一個有效的 cToken
-   `CompoundDebtPositionParser`將要傳出的 100 cDAI 格式化後分別傳入`assetsToTransfer`和`amountsToTransfer`數組(返回到`ExternalPositionManager`)
-   `ExternalPositionManager`將有效負載與資產的數據一起傳遞（100 cDAI）然後接收（none）到`VaultProxy`
-   `VaultProxy`向目標`ExternalPositionProxy`轉移 100 cDAI
-   `VaultProxy`調用`ExternalPositionProxy.receiveCallFromVault()`傳入有效負載
-   `ExternalPositionProxy`調用`VaultProxy`庫其類型的庫（Compound CDP）(庫是從`ExternalPositionManager`查詢到的)
-  `ExternalPositionProxy`使用返回的`CompoundDebtPositionLib`來解析和執行`AddCollateralAssets`操作，將 cDAI 添加到其託管資產的內部會計中，並與 Compound 交互以根據需要使用 cDAI 作為抵押品。

### Staking Wrappers(質押包裝)
一些促進 TokenA 質押的智能合約（例如 OHM）發行 ERC20 TokenB（例如 sOHM 或 gOHM）作為質押存款的收據。作為 ERC20，這些質押收據資產很容易保存在基金庫中，並在贖回股票時轉移給投資者。

其他人本身並不簽發 ERC20 收據，例如 Convex。在這些情況下，需要有一種方法來計算質押頭寸，即一些 TokenA 離開了金庫（它在質押協議中），我們需要知道有多少。雖然這可以通過“外部頭寸”來完成，但這些頭寸仍然具有不可替代性（不可轉讓性等）的缺點，並且將不適用於遵循更嚴格的信任模型的基金。

相反，Enzyme Council 會部署和維護“staking wrappers”(質押包裝)：

質押包裝是：
-   符合 ERC20 的代幣
-   多個存款人的狀態處理程序，包括對每個存款人的質押獎勵的檢查點
-   對任何用戶完全開放，即在 Enzyme 協議之外可以進行存款、取款、轉賬和領取獎勵
-   可以被有限制的暫停（由 Enzyme Council；只有存款和收穫新獎勵是可暫停的；取款和獎勵索賠不可）
-   可升級的代理（ Enzyme Council會提供；僅用於緊急情況）
#### ConvexCurveLpStakingWrapper
Convex Finance 中質押 Curve LP 代幣頭寸的包裝器。
注意事項：
-   一個包裝器部署代表一個特定的質押 Curve LP 代幣，例如 3pool
-   任何一方都可以`ConvexCurveLpStakingWrapperFactory`通過
-   每個Convex 池只有一個官方包裝器部署（由 `ConvexCurveLpStakingWrapperFactory` 強制執行）

### Protocol Fee(協議費)
此版本實施了“協議費”，其本質上是：
-   是對資產管理規模徵稅
-   相對於年化目標百分比（最初為 25 個基點）連續徵收
-   導致燃燒相應數量的 $MLN
#### Approach(方法)
協議費用通過Enzyme Council管理的合同（ `ProtocolFeeReserveProxy`）向鑄造新股的基金收取。每當基金：
-   收到新的存款
-   贖回股份
-   遷移到新版本或重新配置到新版本`ComptrollerProxy`

這種鑄造份額而不是直接收集 $MLN 或其他資產的方法經過了徹底的審查，並產生了最少的用戶摩擦、最精簡的架構和最高的可靠性。

但有一個缺點，即理事會管理的基金`ProtocolFeeReserveProxy`不斷從每個基金中獲得股份，然後需要以某種方式將其轉換為 $MLN 才能被燒毀。通過股票贖回手動執行此操作將是工作密集且效率低的（非常消耗gas），並且在某些情況下甚至是不可能的（例如:所有價值都鎖定在外部頭寸中的基金）。

為了優化費用收取和被動接收 $MLN 進行銷毀的便利性，此版本使用了股票回購機制：

1. 協議費用份額以高於有效目標費率（例如: 25 bps）的膨脹利率（例如 50 bps）生成。
2. 基金可以使用$MLN以折扣價（例如，50%）回購協議收集到的股票。
3. 利用$MLN回購的基金最終支付有效目標利率（25 bps），而沒有（並讓Enzyme Council和協議承擔尋找將股票轉換為 $MLN 的其他機制的負擔）的基金支付膨脹率利率（50 bps）。

還提供了一個核心配置選項，允許資金在收取時自動回購協議費用份額。如果`setAutoProtocolFeeSharesBuyback()`在基金的`ComptrollerProxy`中打開，那麼它將嘗試使用基金`VaultProxy`中可用的 $MLN以原子方式回購所收集的全部協議費用份額（在存款和股票贖回行動期間）。

#### Contracts
為了實現這個機制，兩個不同生命週期和不同關注點的解耦合約被串聯使用：
-   `ProtocolFeeTracker`是一個不可升級的發布級合約，它處理與跟踪每個基金在任何時候所欠股份數量相關的狀態和邏輯
-   `ProtocolFeeReserveProxy`是一個可升級的、持久的合約，作為協議費用份額被鑄造到的存儲庫，其當前`ProtocolFeeReserveLib`處理如何對這些份額採取行動的邏輯，即基金以折扣價回購以換取 $MLN
    
所有對`VaultProxy`上的$MLN 持股（即銷毀）和股票（即鑄造和銷毀）的狀態更改操作都是由`VaultProxy`合約執行的，而不是這些外部合約：
-  `ProtocolFeeTracker`無權調用`VaultProxy`上的鑄造和銷毀份額功能
-  實際上並`ProtocolFeeReserve`合約沒有接收和銷毀自己的 $MLN，這是在`VaultProxy`內部處理的

`ProtocolFeeTracker`和`ProtocolFeeReserve`只有提供`VaultProxy`處理鑄造和銷毀所需的數據。

之所以可行，是因為`ProtocolFeeTrakcer`和`ProtocolFeeReserve`在被`VaultProxy`完全信任的情況下運行，並且這種模式保持邏輯簡單和乾淨，不會將`VaultProxy`方法暴露給額外的狀態更改調用者，並且 gas 使用效率高。
### Gas Relayer
此版本支持通過 [Open GSN ](https://opengsn.org/)中繼交易。

基金經理可以利用這種支持，用`VaultProxy`的WETH 餘額支付允許交易的 gas 費用。

從高層次來看，這是通過為每個基金部署一個符合 GSN 規則的“paymaster”合約來實現的，該合約允許從其關聯的 WETH 中提取 WETH `VaultProxy` 以充值該基金的存款餘額。

#### Architecture
氣體中繼器架構特定於此版本（即，它在版本之間不是“持久的”）。
涉及的主要合同有：
-   `GasRelayPaymasterLib`- 所有“paymaster”實例的規範庫合約，提供與 GSN 合約交互的邏輯，維護健康的 WETH 存款，並定義可以中繼的調用規則
-   `GasRelayPaymasterFactory`- 部署新的“paymaster”（`BeaconProxy`）實例，並且是當前信標庫的參考（即，`GasRelayPaymasterLib`）
-   `GasRelayRecipientMixin`- 所有網關合約繼承的共享邏輯，用於可中繼交易

#### Usage(使用方法)
**使用氣體中繼：**
1. `VaultProxy`中必須有足夠的 WETH來支付當前指定的存款金額`GasRelayPaymasterLib`。
2. 基金持有人呼叫`ComptrollerProxy`上的`deployGasRelayPaymaster()`
3. `ComptrollerProxy`通過`GasRelayPaymasterFactory`部署一個新的“paymaster”（`BeaconProxy`實例）並將 WETH 存入新部署的 paymaster。
4. 基金應保持足夠的 WETH 餘額在`VaultProxy`，以便在存款用完之前補足。

**要關閉 gas 中繼器**，基金所有者可以調用`shutdownGasRelayPaymaster()`，`ComptrollerProxy`將 WETH 存款提取回`VaultProxy`。
**當基金遷移到新版本**時，基金所有者可以調用在“paymaster”上的`withdrawBalance()`將 WETH 存款提取回`VaultProxy`

#### Allowed calls
任何在基金中具有許可角色（所有者、遷移者或資產管理者）的賬戶都可以使用內部氣體中繼器架構。
允許通過以下合約的所有與基金管理相關的調用：
-   `ComptrollerProxy`
-   `VaultProxy`
-   `PolicyManager`
-   `FundDeployer`（僅限重新配置功能；遷移功能無法，因為它們在與付款主管不同的版本上調用）
注意：任何人都可以使用外部“paymaster”來支付中繼交易，並且有意支持啟用存款和贖回股票功能，同時不允許在協議的“paymaster”中進行這些特定調用。
### Policies(政策)
政策旨在提供特殊保證，幫助在當前投資者和基金經理之間建立信任：
-   基金經理的行動
-   當前投資者的行為
-   潛在投資者的行動

政策規則和行為：
-   實施一個或多個“策略掛鉤”，例如，在`PostBuyShares`通過存款鑄造新股後立即調用實施掛鉤的策略來驗證狀態
-   可以定義是否禁用
-   可以定義是否可更新
-   可以隨時添加，如果掛鉤不能反向限制當前投資者的行為_（即，實施的政策`PreTransferShares`或`RedeemSharesForSpecificAssets`掛鉤不能在基金設置/遷移/重新配置之外添加）

#### 政策：潛在的投資者行動
這些政策限制了由誰以及在什麼條件下可以接收新股

##### AllowedDepositRecipientsPolicy
-   鉤子：`PostBuyShares`
-   可禁用：是
-   可更新：是（但列表定義列表項是否可更新）
-   描述：將新存款的接收者限制在地址列表內

##### AllowedSharesTransferRecipientsPolicy
-   鉤子：`PreTransferShares`
-   可禁用：是
-   可更新：是（但列表定義列表項是否可更新）
-   說明：將股份轉讓的收件人限制為地址列表

##### 最小最大投資政策
-   鉤子：`PostBuyShares`
-   可禁用：是
-   可更新：是
-   說明：限制單筆存款的投資金額
-   最大數量 0 可用於禁用所有的新存款

例如，最低 100 USDC 最高 10,000 USDC
例如，最低 100 USDC，沒有最高限額

#### 政策：基金經理行動
這些政策限制了基金經理可能被用來榨取、隱藏價值或超出基金授權範圍的行為
##### AllowedAdapterIncomingAssetsPolicy
-   鉤子：`PostCallOnIntegration`
-   可禁用：否
-   可更新：否（但列表定義列表項是否可更新）
-   描述：限制可以通過適配器操作接收的資產

##### AllowedAdaptersPolicy
-   鉤：`PostCallOnIntegration`
-   可禁用：否
-   可更新：否（但列表定義列表項是否可更新）
-   說明：限制`IntegrationManager`可以使用的適配器
-   預期目的：防止基金經理使用任意適配器。大多數基金應選擇使用理事會維護的已知適配器列表。

##### AllowedExternalPositionTypesPolicy
-   鉤子：
    -   `CreateExternalPosition`
    -   `ReactivateExternalPosition`
-   禁用：否
-   可更新：否
-   描述：通過阻止將外部頭寸添加到保險庫，限制可以使用的外部頭寸“類型”（例如，Compound CDP）。
-   預期目的：主要目的是完全防止經理使用外部部位，儘管它也可以用來限制允許的外部職位種類
例如，不允許有外部頭寸
例如，僅允許使用Compound CDP

##### 累積滑點公差政策
-   鉤子：`PostCallOnIntegration`
-   可禁用：否
-   可更新：否
-   描述：限制在“容限期”（7 天）內通過適配器操作可能發生的價值損失（即滑點）。資金定義了自己的容忍量（例如，5%、10% 等）。當適配器操作導致滑點時，該滑點量被添加到累積滑點總數中。然後，累積滑點會在“容忍期持續時間”內以基於基金選擇的容忍度的恆定速率減少。如果被調用的適配器在理事會維護的列表中`AddressListRegistry`（資產經理無法操縱以竊取資金價值的適配器），則此策略允許完全繞過滑點檢查。
-   預期目的：減慢惡意經理耗盡資金以允許警報和退出的速度

例如，具有 10% 容差的基金可能在任何一筆交易中遭受 10% 的最大滑點，然後在 7 天的容差期結束之前，補足金額的最大滑點 (10% * secondsPassed / oneWeekInSeconds)。

##### OnlyRemoveDustExternalPositionPolicy
-   鉤子：`RemoveExternalPosition`
-   可禁用：否
-   可更新：不適用（無設置）
-   描述：允許從保險庫的 `activeExternalPositions`中移除外部部位，僅當其值可以忽略不計時。閾值由理事會維持。該政策允許將沒有有效價格的外部頭寸的適當信號標的資產估值為`0`
-   預期目的：防止經理在未跟踪的外部頭寸中隱藏重要價值，同時允許刪除可忽略不計的頭寸，這些頭寸計入金庫`POSITIONS_LIMIT`並增加大量 gas 成本以資助功能
    

##### OnlyUntrackDustOrPricelessAssetsPolicy
-   鉤子：`RemoveTrackedAssets`
-   可禁用：否
-   可更新：不適用（無設置）
-   描述：僅當 a) 資產沒有有效價格或 b) 其價值可忽略不計（即灰塵）時，才允許從保險庫中移除資產。`trackedAssets`塵埃閾值由理事會維持。
-   預期目的：防止經理將重要價值隱藏在金庫中作為未跟踪資產，同時允許移除可忽略不計的價值頭寸，這些頭寸計入金庫`POSITIONS_LIMIT`並為資金功能增加大量gas成本，還允許移除價格無效的資產阻止存款和其他功能。

#### 政策：當前的投資者行為

這些政策防止投資者有意或無意地擾亂基金策略和流程

##### AllowedAssetsForRedemptionPolicy
-   鉤子：`RedeemSharesForSpecificAssets`
-   可禁用：是
-   可更新：否（但列表定義列表項是否可更新）
-   說明：經理定義了允許包含在特定資產贖回中的資產
例如，不允許任何資產（即不允許特定資產贖回）
例如，只允許 WETH 和 MLN

##### MinAssetBalancesPostRedemptionPolicy
-   鉤子：`RedeemSharesForSpecificAssets`
-   可禁用：是
-   可更新：否
-   描述：經理定義了特定資產贖回後必須保留在保險庫中的最低資產餘額
-   預期目的：通過維持最低餘額來保證氣體中繼器（WETH）和自動股票回購（MLN）的持續功能

例如，每次贖回後，至少 1 WETH 和 10 MLN 必須保留在保險庫中

### Asset pricing(資產定價)
酶保險庫及其份額（以及某些政策和費用）依賴於無需信任地計算所持有資產的價值。
這種必要性為協議創建了一個有界的“資產宇宙”，其中資產必須具有合理、可靠的鏈上定價機制，然後必須由酶委員會添加到協議中。
通常，如果資產有可用的 Chainlink 提要，則直接從那裡查詢其價格（通過[`ChainlinkPriceFeed`](https://specs.enzyme.finance/external-interactions/price-feed-sources#chainlinkpricefeedmixin)）。
否則，必須創建定制的酶價格饋送，以便為資產類別制定定價機制。

#### Pricing risk(定價風險)

如果投資者是未知/不受信任的實體，基金所有者和資產管理人必須了解他們所持有的資產所涉及的定價機制假設和漏洞。這是因為如果 Enzyme 對其所持資產價值的理解與通過交易或贖回這些資產實際可以獲得的價值存在偏差，則基金可能會面臨股價套利。（請參閱“[已知風險和緩解措施](https://specs.enzyme.finance/topics/known-risks-and-mitigations)”）。

#### 示例：使用 Chainlink 價格的包裝或合成資產

對於許多封裝資產（例如 wBTC、cxDOGE 和 Lido stETH）和潛在的合成資產（目前沒有），該協議假設資產（例如 wBTC）與其標的資產（例如 BTC）保持 1:1 的價格。
在包裹資產的情況下，標的資產由第三方（無論是 EOA 還是合約）保管。如果託管實體丟失了對資產的訪問權（例如，合約漏洞或私鑰洩露），協議將繼續將包裝器與其底層的 1:1 視為 1:1，即使它的實際價值在 1 和 0 之間。
抵押不足的抵押合成資產也是如此。

#### 示例：依賴外部協議假設的資產

例如，[`CurvePriceFeed`](https://specs.enzyme.finance/external-interactions/price-feed-sources#curvepricefeed)如果池中的任何資產失去與池不變的 1:1 掛鉤（這將導致各種“銀行擠兌”，使曲線池失衡），用於定價已抵押和未抵押的 Curve 池代幣將變得不穩定.

#### 示例：依賴外部協議安全的資產
例如，[YearnVaultV2PriceFeed](https://specs.enzyme.finance/external-interactions/price-feed-sources#yearnvaultv2pricefeed)用於為 yVault 代幣定價的 yVault 合約依賴於其 yVault 合約以至於無法被價格預言操縱攻擊操縱的方式正確報告其價值。

### Known Risks & Mitigations(已知風險和緩解措施)

就各種行為者的行為而言，主要有兩種風險類別：
-   機會主義的投資者
-   機會主義的管理者（包括所有者、遷移者和資產管理者）

不同的基金設置將在這些各方內部和之間具有不同程度的信任。

例如，DAO 基金可能只有一個投資者，表面上與所有者是同一實體。或者，他們可能會將資產管理委託給只應在特定參數內運行的 EOA。
例如，個人基金所有者可能是知名人士，其聲譽對投資者來說是一種自然的緩解措施。或者，他們可能是完全匿名且不受信任的。

為了不對所有基金全面應用最嚴格的風險緩解措施，該協議在很大程度上對什麼構成“安全”設置沒有意見，但提供了各種配置選項和策略來製作滿足特定信任要求的定制設置。

#### 機會投資者（套利）
投資者可以通過在合適的條件下從基金中存入和/或贖回基金持有的暫時錯誤定價的股票或錯誤定價的資產來套利。
_針對以下機會的一般緩解措施：_

-   `sharesActionTimelock`配置選項定義了用戶最近的存款和他們的下一次轉賬或贖回之間必須經過的秒數。雖然 1 秒足以防止閃現和三明治漏洞利用，但 1 秒越長`sharesActionTimelock`，套利機會在允許的贖回時間保持開放的保證就越少。
-   當存入或贖回（有條件地）扣除一定數量的股份時，可能會產生費用，從而提高其有效股價。例如，為了減少存款套利，希望使用額外套利保護的基金可以使用`EntranceRateBurnFee`，它會燒掉存款期間鑄造的 a%(可以自行設定) 股份。

##### 機會：由於存款期間價值未追踪而導致股票定價錯誤
可能存在不包含在其股價中的“屬於”基金的價值，即“未追踪資產” `VaultProxy`（例如：空投）和無人認領的“外部獎勵”（例如，應計 `COMP` 獎勵）
_額外的緩解措施_：
-   通過協議獲取到 VaultProxy 的新資產會自動添加為跟踪資產
-   管理人員應盡快跟踪任何未跟踪的資產（例如，通過空投或公開調用的獎勵索取功能）
-   在累積外部獎勵（例如 COMP）的情況下，建議經理跟踪他們希望盡快獲得的任何獎勵代幣，例如，一旦您第一次通過 Compound 借貸，就立即跟踪 COMP。
    

##### 機會：由於存款期間的鏈上價格導致股票定價錯誤

由於獨家使用鏈上資產價格，可能會出現資產鏈上鍊下價值出現偏差的情況，進而導致股價出現偏差。

儘管協議中使用的價格反饋應該被認為是抗操縱的，但仍然存在通過交易更新價格的情況，因此可以提前運行（例如，Chainlink 聚合器價格更新）。此外，價格更新可能僅在超過偏差閾值（即 Chainlink 聚合器）時發生，即使在足夠大的基金中小於 1% 的嚴格閾值也可能導致大量套利機會。

如果股價“太低”（即，根據鏈上內部使用價格計算的資產總價值低於根據規範價格計算的總價值），那麼新投資者可以存款並獲得折扣。

##### 機會：特定資產贖回時因鏈上價格導致資產定價錯誤

同樣，如果內部使用的資產價值偏離其標準價值，`redeemSharesForSpecificAssets()`贖回選項可以通過撤回相對於基金中的其他資產定價“過低”的一項或多項資產來進行套利。

雖然使用`sharesActionTimelock`會阻止尚未持有股票的用戶行使這種套利機會，但時間鎖定已到期的當前投資者可以隨時贖回。

_額外的緩解措施：_
-   僅針對`FeeHook.RedeemSharesForSpecificAssets`（即，不針對實物贖回）收取的退出費，該費用消毀a %了正在贖回的股份的百分比。

#### 機主義基金經理

除了故意將基金暴露於套利的機會主義行為（即不遵循上述建議）之外，基金經理（所有者、遷移者和資產管理人）還可以通過錯誤的配置或持有的不良行為直接從基金中竊取價值。

這就是為什麼投資者必須評估自己對基金所有者（任命遷移者和資產管理人角色）的信任門檻以及基金是否針對該門檻進行了充分配置（核心配置、費用和政策）。

協議中管理者風險緩解的一個關鍵概念是，在不過度限制管理者行為的情況下，要完全阻止管理者竊取價值是極其困難的。目標是減緩價值竊取，讓投資者有足夠的時間注意到（或被通知）並在必要時退出基金。

還需要注意的是，資金可以在重新配置時間鎖的範圍內隨時重新配置（即新政策、費用和設置）。

##### 機會：通過適配器耗盡資金

有可能通過一些適配器以機會主義的方式進行交易，導致價值從基金洩漏到外部賬戶（即，到基金經理）。
例如，Uniswap 上的多跳交易可以通過任意中間池進行路由，其中管理者是唯一的 LP 提供者。通過 ParaSwap 上的各種途徑，類似的利用是可能的。
緩解：使用限制在給定時期內允許的股價價值損失的政策，例如，24 小時內 5%

#### 機會：隱藏外部頭寸的價值
沒有從外部頭寸清算和強行收回價值的機制。由於外部頭寸位於金庫之外並且在贖回實物時不包括在內，因此經理可以將部分或全部 GAV 轉移到外部頭寸中，從而有效地保護其免受任何贖回嘗試。一旦被扣為人質，經理可以永久收取管理費，持有資產以勒索贖金，或對基金進行惡意的重新配置。

允許存在於外部頭寸中的 GAV 百分比越高，對投資者來說就越危險。

緩解措施：
-   完全不受信任的基金可以通過政策完全排除使用外部頭寸
-   擁有更多信任或風險承受能力的投資者的基金可以使用限制跨外部頭寸允許的 GAV 百分比的政策
    

#### 機會：取消跟踪資產

經理可以取消跟踪基金中的任何跟踪資產（面額資產除外），從而有效地暴露上述股票套利機會。

緩解措施：
-   使用將刪除跟踪資產限制在可忽略不計的數量的政策
    

#### 機會：任意費用、政策和適配器

默認情況下，管理員可以使用任何費用、策略或適配器，它們可以包含任意邏輯，可以通過 CREATE2 秘密升級等。
最終，投資者必須信任基金的設置。
費用減免：
-   費用只能在基金設置/遷移/重新配置時添加
政策緩解：
-   限制用戶 (`PolicyHook.RedeemSharesForSpecificAssets`和`PolicyHook.PreTransferShares`) 的策略只能在設置/遷移/重新配置時添加

適配器緩解措施：
-   使用僅允許 Enzyme Council 信任的適配器的策略
## Fee Formula(費用公式)

### ManagementFee(管理費)
#### definitions
管理費率（年度，百分比）：`x`
有效管理費率（年度，百分比，稀釋後）：`k`(新股增發比例)
由於管理費不支付（作為資產的百分比），而是分配為基金中的新股，我們需要使用有效管理費率。這可確保經理收到正確的股份比例。
兩種費率的關係如下：
The two fee rates are related as follows:

$x = \frac{k}{1+k}$

或者

$k = \frac{x}{1-x}$

#### Continuous compounding(連續複利)
管理費應計發生在不定期且未知的時間間隔內，因此我們必須訴諸連續複利。持續管理費率`z`與年度有效管理費率`k`的關係如下：
$e^z=1+k$
或者
$z = ln(1+k)$

代入有效管理費率`k`，得到連續管理費率與年度管理費率的關係：
$e^{z} = \frac{1}{1-x}$
或者
$z=-ln(1-x)$

#### Management fee allocation(管理費分配)
每當管理費在一段時間後到期`t`（表示為一年的一小部分），股份數量變化如下
`S`：是分配管理費股份前的股份總供應量
`S'`：是分配管理費股份後的股份總供應量。
$S' = e^{z\cdot t} S$

分配給經理的股份為 `S_{manager} = S'-S`，計算如下：

$S_{manager}​=(\frac{1}{(1-x)^t}−1)S$

或者

$S_{manager}​=((1+k)^t−1)S$

使用 `t = \Delta t / N`，我們可以將其重寫為
$S_{manager}​=(f^{Δt}−1)S$
$f = (1+k)^{1/N}$

`f`在配置費用時在鏈下計算，並在鏈上存儲為`scaledPerSecondRate`. 然後是鏈上計算

`sharesDue = (rpow(scaledPerSecondRate, numberOfSeconds, 10*27) - 10**27) * totalSupply / 10**27`
### Performance Fee(績效費)
 Sulu 前版本的性能費用使用“結晶期(Crystallisation Period)”的概念。這個概念在傳統金融中很重要，但要正確實施在鏈上很複雜且成本很高。
通過去掉“結晶期”的概念，我們可以大大簡化協議中績效費的實現。
這也意味著可以隨時索取績效費。
如果沒有“結晶期”，經理可能會通過持續累積而不是每季度或每年累積來賺取更多績效費。因此，管理人員應將新簡化績效費的費率設定為低於先前使用的績效費的費率。

#### 原則#
-   績效費是在一段時間的持續股份供應後支付的。在以下動作中股份會產生變化：
    -   購買份額
    -   贖回份額
    -   獲取費用
-   只有在股票期末的股價高於高水位線時，才會支付業績費。
-   只有股價高於高水位所創造的財富才是(?)
-   與所有其他費用一樣，績效費以股票形式支付。
-   績效費需在管理費後登記（即管理費需先計算，管理費分成），但必須
-   費用登記順序：管理費、績效費、入場費、退出費

#### 公式
- 調用`totalSupply` =$TS_i​$ (在鑄造或銷毀前的所有份額)
- 從 storage 讀取`highWatermark`(這是上一次業績費計算後的股價)$hwm$
-   當前總股價$g_i = GAV_i / TS_i$
-   期間創造的財富:$W_i = max(g_i - hwm, 0) \cdot TS_i$
-   期間績效費的價值$F_i = W_i \cdot x$
	-  $x$是績效費百分比
-   績效費份額（稀釋現有份額）:$f_i = \frac{F_i \cdot TS_i}{GAV_i - F_i}$
-   計算股價（在所有費用都被鑄造或燒掉之後）:$g_i^\prime = GAV_i / TS_i′$ 
	- $TS^\prime_i$是所有費用結算後的新總供應量
	- 如果$g^\prime_i > hwm$($W_i$和$F_i$將會大於0)就更新storage $hwm = g^\prime_i$

## External interaction
### Adapters
為了將基金的一些資產交換為其他資產，適配器通常與一個或多個“集成者”集成，即與 #### AAVE
- lend
- redeem

#### Compound
- lend
- redeem
- claimRewards

#### ConvexCurveLpStaking
- claimRewards
- stake
- unstake
- lendAndStake
- unstakeAndRedeem


#### CurveExchange
- takeOrder

#### CurveLiquidity
- claimRewards
- lend
- redeem
- stake
- unstake
- lendAndStake
- unstakeAndRedeem

#### Fuse(rari capital)
- lend
redeem
claimRewards

#### Idle
approveAssets
claimRewards
lend
redeem

#### OlympusV2
stake
unstake

#### ParaSwapV5
takeOrder

#### PoolTogetherV4
claimRewards
lend
redeem

#### Synthetix
takeOrder
redeem

#### UniswapV2Exchange
takeOrder

#### UniswapV2Liquidity
lend
redeem

#### UniswapV3
takeOrder

#### YearnVaultV2
lend
redeem

#### ZeroExV2
takeOrder

### External Positions(外部部位)
外部部位與外部協議交互，就像適配器一樣。通過`ExternalPositionProxy`實例​​及其庫的交互會影響其值，而通過解析器合約的交互只會影響驗證和用戶輸入的解析。
#### External Position Types
每個外部部位都與`ExternalPositionProxy.EXTERNAL_POSITION_TYPE`中的“類型”相關聯(在部署時就確定)。這種類型(例如，Compound debt position)指示系統使用哪個庫和解析器(parser)合同與代理交互。

一個基金可以擁有許多同類型的外部部位。

沒有任何全局強制的規則在外部部位的資產上。如果對於特定外部部位類型需要限制，那麼解析器合約應該對該類型施加限制。
##### AaveDebtPosition
##### CompoundDebtPosition
##### UniswapV3LiquidityPosition
### Price Feeds
正如適配器和外部部位與外部“整合者”交互一樣，價格反饋也與為它們提供提供費率所需數據的來源進行交互。
#### AavePriceFeed
Aave `aTokens`的價格，與底層證券價值 1:1 掛鉤，例如 1`aAAVE`等於 1 `AAVE`。`aTokens`通過公式化的變基機制產生利息。
注意事項：始終直接將輸入金額作為輸出金額返回（即1:1掛鉤）
#### ChainlinkPriceFeedMixin
每個匯率對（我們使用以美元或 ETH 報價的匯率）由 Chainlink “聚合器”提供，我們與這些聚合器的代理合約進行交互。
注意事項：
我們不會在每次價格查詢時檢查時間戳，而是提供一個函數來允許快速刪除被認為是陳舊的聚合器。如果一個聚合器被移除，相應的基礎資產將停止從提要中產生價格並將恢復，導致任何依賴該價格的函數失敗，直到它被重新建立。這是期望的行為。
#### CompoundPriceFeed
直接查詢每個 Compound Token (cToken) 的費率。
主網合約：每個 cToken
注意事項：
我們查詢緩存費率而不是實時費率以節省氣體。cTokens 費率在很長一段時間內變化可以忽略不計。
#### ConvexCurveLpStakingWrapperPriceFeed
將一定數量的`ConvexCurveLpStakingWrapper`代幣轉換為其基礎 Curve LP 代幣（比率始終為 1:1）
#### CurvePriceFeed
提供所有 Curve LP 代幣的價格，包括質押和非質押。將 LP 代幣質押到池的“流動性指標(liquidity gauge)”（或流動性指標包裝合約）會返回等量的代幣，代表該流動性指標中的股份。即 1 個 LP 代幣的價值相當於 1 個流動性計量代幣。

請記住，在 Enzyme 中導出衍生品的基礎價值包括返回基礎資產和金額，返回值有兩個組成部分：
1. 基礎資產：每個 LP 代幣（質押或非質押）被任意分配一個“不變代理資產”，協議中支持的資產被選為礦池標的代理（例如， WETH-> stETH 池）。
2. 基礎數量：不變代理資產的金額是通過池的“虛擬價格”計算的（在所有 Curve 池中原生提供）。

注意事項：
由於為價格反饋返回的單一標的資產選擇了任意資產（而不是考慮池標的的所有價值和余額），酶中 LP 代幣的實時價值將始終與可贖回價值偏離一小部分. 由於以下幾個原因，這種偏差是微不足道的：
-   暫時失去錨定的資產通常會向下失去，這將導致 LP 代幣的可贖回價值低於其虛擬價格衍生的 Enzyme value。這種情況不易受到 Enzyme 基金的投資者套利的影響，其中擔心的是阻止購買折扣股票（而使用虛擬價格會導致股價暫時上漲）。此外，只要假定掛鉤的損失是暫時的，使用虛擬價格而不是這種短暫的不平衡可以保護基金免受折扣股票的影響。
-   在更極端的永久失去掛鉤的情況下，Curve 僅在所有集合資產通常都與不變量掛鉤的情況下才有效。如果任何一項資產永久失去與不變量的掛鉤，LP 代幣本身的可贖回價值就會投降，因為套利者會耗盡除下跌資產之外的所有資產。
#### FusePriceFeed
直接查詢每個 Fuse Token (fToken) 的比例
注意事項：
查詢緩存費率而不是實時費率以節省氣體。fTokens 費率在很長一段時間內變化可以忽略不計。
#### IdlePriceFeed
為每個`IdleToken`提供其基礎資產的價值。
注意事項：未考慮在贖回`IdleToken`其底層證券時收取的特定於用戶的費用（即，相對於保險庫）
#### PoolTogetherV4
價格 PoolTogether `ptTokens`，與底層證券價值 1:1 掛鉤，例如 1`ptUSDC`等於 1 `USDC`。
#### RevertingPriceFeed
在任何價格查詢時立即恢復。
的目的`RevertingPriceFeed`是能夠將資產保留在我們的資產宇宙中，同時禁用任何依賴於準確 GAV 計算的操作。它僅用作臨時的權宜之計，以允許集成的持續功能，但以核心系統中的有限功能為代價。
跟踪使用 RevertingPriceFeed 的資產會導致依賴於 GAV 的操作恢復，例如：
-   存款
-   取決於 GAV 的費用
-   依賴於 GAV 的政策
請注意，在股票贖回期間會跳過恢復費用，這將影響具有在贖回時結算的 GAV 相關費用（例如`PerformanceFee`）並跟踪使用 RevertingPriceFeed 的任何資產的基金。

#### UniswapV2PoolPriceFeed

使用特殊的防池操作公式，該公式考慮了給定 Uniswap 池 ( `UniswapV2Pair`) 中當前的基礎資產餘額和池代幣餘額以及兩個基礎代幣之間的可信比率。
另請參閱我們基於此的示例實現：https://github.com/Uniswap/uniswap-v2-periphery/blob/267ba44471f3357071a2fe2573fe4da42d5ad969/contracts/libraries/UniswapV2LiquidityMathLibrary.sol
#### YearnVaultV2PriceFeed
為每個（Yearn v2）的基礎資產提供一個值`yVault`
注意事項：
-   實時價格
-   請注意，雖然所有 Yearn v2 實例都遵循相同的接口，但每個單獨的實例都使用許多特定版本的實現之一（請參閱上面的註冊表合同）。只有與接口函數的預期行為的價格反饋交互才能被實際審計。

## Governance 
### Governance Overview
Enzyme 治理模型是以用戶為中心的模型。它確保用戶可以無須許可地訪問安全的資產管理協議，並避免受到網絡中惡意行為者的影響。同時，用戶可以選擇從協議之上的持續創新和改進中受益，並受到受信託責任約束的酶委員會的徹底檢查和分析的保障。Enzyme Council（將在下一節詳細介紹）負責做出維護網絡用戶利益的決策。

用戶始終保持完全控制，並且是他們正在運行的軟件的唯一決策者。

Enzyme Council 和代幣持有者都不能影響基金經理使用的智能合約代碼。基金經理必須自願採取行動才能升級到新版本的代碼，如果基金的投資者對正在使用的代碼版本不滿意，他們可以自由地立即贖回他們的股票。基金經理永遠不會被迫使用他們可能會或可能不會感到舒服的新版本代碼。用戶對從可能包含安全漏洞的代碼進行升級承擔全部責任。

因此，用戶對特定版本的 Enzyme 協議的趨同應向酶委員會強烈表明他們與用戶的情緒和需求保持一致。儘管Enzyme Technical Council（ETC）擁有並控制著指向最新合約的 ENS 子域，但用戶才是真正決定其業務基於哪個版本的人，這對社區構成了強烈的信號。智能合約不可停止的特性進一步實現了這一點（一旦部署，Enzyme 合約就不能被部署者收回）。

但是，我們強烈建議用戶始終使用最新版本的 Enzyme 協議，因為可以發現安全漏洞並將在協議升級中修復。項目方也鼓勵用戶對他們打算使用的合同進行自己的分析、審計和審查。最終的選擇和責任完全取決於用戶。

### The Enzyme Council
Enzyme 協議最初是由一家名為 Melonport AG 的瑞士公司開發的。在 2019 年 2 月協議主網上線後，Melonport 結束了運營，協議的治理權移交給了酶委員會。該委員會由酶技術委員會 (ETC) 和酶用戶代表 (EUR) 的代表組成，這兩者將在後續章節中介紹。

**受託責任和利益衝突**

Enzyme Council 會受信託責任、指導原則和 Enzyme Council 章程的約束。這意味著酶委員會成員有義務為酶協議的最佳利益行事。任何違反其受託義務的成員都將面臨撤銷其席位的風險。
如果酶委員會成員在特定問題上存在利益衝突，他們應立即通知其他成員，並對相關事項投棄權票。

#### 領導和組織

酶委員會設有主席和副主席，每兩年輪換一次。他們的職責包括協調會議和議程。酶委員會還包含幾個在特定主題上發揮領導作用的小組，例如：審計、功能、生態系統項目、網絡參數、代幣經濟學、社區電話等。
**今天誰在酶委員會任職？**
-   **Chain Security (ETC)**
-   **Exa:** Co-founder of Exponent (ETC)
-   **Janos Berghorn:** Investor @ KR1 (ETC)
-   **Giel Detienne**: User representative (EUR)
-   **Mona El Isa**: Founder & CEO @ Avantgarde Finance (ETC)
-   **Felix Hartmann**: Founder @ Hartmann Capital & User (EUR)
-   **Will Harborne**: Founder & CEO @ Deversifi (ETC)
-   **Nick Munoz-McDonald**: Smart Contract Auditor & Researcher @ G0 Group (ETC)
-   **Paul Salisbury**: Founder @ Blockchain Labs (ETC)
-   **Zahreddine Touag:** Founder @ Woorton (ETC)
-   **Theophille Villard**: Co-founder of Multis wallet, contributor to Enzyme codebase (ETC).
#### **激勵結構**
具有 ETC 合適技能的人很少。因此，成員受到協議年度通貨膨脹的一部分的激勵。該部分目前的上限為 20%，並按 2 年的時間表歸屬。

最初，這應該只涵蓋 ETC 的成本，但通過將獎勵作為年度通脹的百分比（即總市值的比例）提供，我們還引入了通過為網絡增加價值來增加 Enzyme 市值的激勵措施。

申請EC
可以通過發送電子郵件至council@enzyme.finance 申請ETC 和EURs。

https://specs.enzyme.finance/governance/the-enzyme-council/the-enzyme-technical-council-etc