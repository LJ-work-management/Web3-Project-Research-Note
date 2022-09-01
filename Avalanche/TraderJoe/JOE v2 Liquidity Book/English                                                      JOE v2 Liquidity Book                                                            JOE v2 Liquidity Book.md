original article: https://github.com/traderjoe-xyz/LB-Whitepaper

## Abstract
We introduce Liquidity Book (LB), a novel protocol for structuring liquidity in a decentralized exchange. The LB protocol enables the creation of unique and dynamic liquidity structures for a paired asset market. Liquidity is pooled in constant price bins which are aggregated to establish a market.

## 1 Introduction

Trader Joe (v1) was launched in 2021 as an AMM with Uniswap v2 liquidity contracts as its backbone . As DeFi advances, we have concluded that capital efficiency and liquidity flexibility are urgent issues that demand protocol development. In this paper, we present our design for the next stage of the JOE decentralized exchange.

### 1.1 Concentrated Liquidity

Tick bound liquidity, as implemented in Uniswap v3 (v3), pioneered the concept of concentrated liquidity for a general automated market maker (AMM) . In Uniswap’s implementation, individual liquidity positions are established which provide reserves over a restricted price range. The reserves of each position are exchanged over its price range following a common constant function formula. This common function allows individual position reserves to be aggregated together to form the market.

Even before the concentrated liquidity term was coined, StableSwap/Curve provided a similar solution for stable exchange rate reserves. Curve v1 utilizes an invariant curve with an implied static peg . While the liquidity is unbounded, the bulk of reserves concentrate liquidity around the peg price. Curve v2 introduces a more general solution using an internal price oracle to upgrade to a dynamic peg .

Constant function algorithms used in Uniswap and Curve restrict the ability of liquidity providers to deploy tailored market making strategies. We propose a new protocol with Discretized Concentrated Liquidity to provide a flexible infrastructure layer for market participants. Discretized Concentrated Liquidity has been previously presented by the iZiSwap limit order implementation .

### 1.2 Liquidity Book

The Liquidity Book (LB) arranges liquidity of an asset pair into discrete price bins. Reserves that are deposited into a liquidity bin are available to be exchanged at the constant exchange rate (price) that is defined for the bin. The market for the asset pair is constructed by aggregating all of the discrete liquidity bins. For example, a market liquidity structure supplied by different market participants (color coded) could be represented by the figure below.

![](https://i.imgur.com/CQPjew0.png)
Figure 1: Example Liquidity Structure

## 2 Protocol Design
### 2.1 Reserve Pricing
Following AMM convention, price (P) is defined as the rate of change in Y reserves per change in X reserves. Price represents the value of the base asset (X) denominated in the quote asset (Y) terms.
$$ 𝑃 = ∆y/∆x$$

## 2.2 Bin Liquidity
Liquidity (L) is defined to be the nominal value of reserves deposited into a discrete bin and is denominated in the quote asset. Liquidity is conserved as the bin’s reserve composition changes and can be described in Eq 2.2 as the value sum of its composite assets.
$$ 𝑃_{j}*𝑥 + 𝑦 = L $$
每個單獨的 bin 是一個具有自己獨特的聯合曲線的恆和市場。 bin 的曲線完全由 bin 的價格參數（靜態）和存入 bin 的總流動性儲備（動態）定義。如圖 2 所示， y 儲備截距表示 bin 的總流動性 (L)，而斜率由 bin 的價格 (𝑃 ) 定義。
![](https://i.imgur.com/Vj6NCYB.png)
圖 2: 儲備聯合曲線

### 2.3. 箱組成
由於上述設計，流動性箱中的儲備組成必須獨立於價格和流動性。可以使用一個附加變量來描述箱中可用的儲備。這個變量，組成因子（c），將被定義作為當前作為 Y 儲備持有的 bin 流動性的百分比。組成 (c) 以 [0,1] 為界。
$$ c≡y/L $$
(Eq: 2.3)
每個流動性箱的資產儲備可以用箱流動性 L 和成分 c 來充分描述，如公式 2.4 和 2.5 所示。
(Eq: 2.4)
$$ y=cL $$
(Eq: 2.5)
$$ x=L/P(1-c) $$

流動性箱中 X 和 Y 儲備的示例組合可以是
可視化，如圖 3 所示。
![](https://i.imgur.com/WxV73qn.png)
圖 3: 箱儲備部位(P14)

### 2.4. 市場聚合
一個資產對的單個流動性箱的聚合建立了該對的市場。流動性箱的儲備狀態預計會隨著市場定價的變化在 X 儲備和 Y 儲備之間轉換。
定價低於市場的流動性箱將僅包含資產 Y，而定價高於市場的流動性箱將僅包含資產 X。
因此，市場價格被確定為包含 X 儲備的最低價格箱。

![](https://i.imgur.com/5k6HT19.png)
圖 4: 聚合流動性部位

### 2.5. 做市
借助離散的流動性箱，流動性提供者能夠創建自定義流動性結構，使他們能夠根據各自的目標、預測和風險狀況輕鬆管理自己的頭寸。
流動性提供者可以將流動性集中在某些價格點或點差附近，並調整各個流動性箱，而不會影響其在其他箱中的流動性定位。總體而言，流動性賬簿設計允許對做市商成功運作所需的範圍和深度進行微調。

## 3 協議架構
3.1 市場定義
每個市場的價格範圍都被限制在2 的正負 128 次方之間
根據稱為 bin step (s) 的離散化參數將此價格範圍離散化為 bin。
一個獨特的市場由 (X, Y, s) 集合定義。
給定市場中流動性箱的總數可以描述為 $$ 2*𝑁_{𝑏} $$ ，其中 $$ N_b $$ 使用公式 3.1 計算。Nb 表示價格平價 (par) 和市場中最大價格之間可以存在的最大箱數給定的 bin 步驟。
(Eq: 3.1)
$$ 𝑁_{𝑏} = 𝑎𝑟𝑔𝑚𝑎𝑥_{i}(1 + 𝑠)^i < 2^{128} $$

除了價格限制外，最大 bin 數量限制為小於 2^24
在具有最大箱數的市場中，將一半的潛在箱分配給小於 par 的值，將一半分配給大於 par 的值會導致 par 箱標識符為 2^23，它分配給公式 3.2 中的常數 b在所有市場中，倉位將被分配一個正整數標識符 (i)，其範圍在 [b - Nb, b +Nb] 之間。
$$ c≡B^{23} $$

### 3.2 倉價
為每個 bin 規定的價格是該對的 bin 步長 (s) 和 bin 的標識符 (i) 的函數，如公式 3.2 所示。 bin 步長參數確定每個增量 bin 之間價格的恆定百分比增加或減少。

(3.3) 
$$ 𝑃_𝑖 = (1 + 𝑠)^{𝑖−𝑏} $$

### 3.3 流動性追踪
為了優化流動性跟踪，箱在通過嵌套三個 256 位數組創建的樹中進行索引，如圖 5 所示。在這棵樹中，每個箱都被分配一個通過嵌套數組的位置路徑。非零值分配給三個表示 bin 位置的數組元素。使用此樹，可以在交換期間有效地搜索市場的流動性狀態和/或從外部監控
![](https://i.imgur.com/56yDag8.png)
圖 5: 箱子指數樹

### 3.4 貿易路線
LB 路由器將為掉期和流動性存款/取款提供額外的安全性和滑點檢查。
當找到更好的定價時，掉期也將通過傳統的 AMM 對進行路由。

### 3.5 流動性代幣
專門為 LB 協議開發了一種新的流動性代幣標準，以支持每個市場中的大量 bin。通過設計，LB 市場中的流動性 (L) 可以跨 bin 組合，因為它通常以 Y 值計值並且是獨立的垃圾箱的狀態。
因此，流動性收據代幣 LBToken 將攜帶相等的流動性值，而不管它包含在哪個 bin 中。此屬性使 bin 流動性能夠跨 bin 無縫捆綁在一起。LBToken 本質上類似於傳統 ERC-20 流動性代幣，具有以下特點關鍵區別：它們在鑄造時被分配了一個 id
匹配流動性所在的 bin 標識符 (i)。
LBToken 合約遵循 ERC-1155 多代幣標準，但僅限於包含可替代的 ERC-20 代幣。
LBToken 是可替代的、以價格為定位的流動性收據代幣。

### 3.6 市場參數
每個市場都會有一組參數來管理市場的費率，在第 4.3 節中進一步詳細說明。初始參數值將根據為市場選擇的 bin 步參數分配。
市場只能通過在工廠中具有相關費用參數的 bin 步驟來建立。
費用參數可以在初始化後進行調整。

![](https://i.imgur.com/bFBHV3q.png)

### 3.7. 預言機
LB 市場將有一個預言機，將以下值記錄到
區塊鏈：
- 時間戳
- 累積ID
- 累積累加器
- 累計 Bin Crossed

累積值對於確定隨時間變化的變化很有用。預言機提供市場價格和波動率/費用指標，因為它是算法交易和流動性提供的有用信息。
sampleLifetime 設置（採樣率）是市場創建時定義的市場參數，用戶可以根據需要增加 Oracle 樣本量。

## 4 用戶交互
### 4.1 添加/刪除流動性
向活躍流動性箱添加和移除流動性將保存價格 (P) 和成分 (c)。對於給定的流動性調整，所需的儲備 X 和儲備 Y 可以使用下面的公式 4.1 和 4.2 計算。
(4.1)
$$∆𝐿 =𝑃/(1−𝑐)*∆x$$
(4.2) 
$$ ∆𝐿 = 1/𝑐*∆y$$

對於所有“未啟動”(不在價格上)的流動性箱，流動性可以專門添加到儲備 X 或儲備 Y 中，具體取決於組合邊界。

(4.3)
$$ ∆𝐿_{𝑐=0} = 𝑃*∆x $$


(4.4)
$$ ∆𝐿_{𝑐=1} = ∆y $$

如果比例不等於 bin 的組成，添加到活動 bin 的流動性將自動在儲備之間進行交換，並且交換的數量將產生交換費用。
將流動性添加到與 bin 的組成不一致的“未啟動” bin 的交易將失敗。

### 4.2 Swap
在流動性箱內交換儲備將保存流動性和價格，因此只有儲備組成會改變。儲備按照公式 4.5 進行交換，而組成保持在 [0,1] 的範圍內。
(4.5) 
$$ Δ𝑦 = − 𝑃Δ𝑥 $$

在達到組合界限之前，流動性箱中可用的儲備可以根據公式 4.6 和 4.7 計算。
(4.6)
$$ Δ𝑐 =−𝑃/𝐿*Δ𝑥
$$
(4.7) 
$$ Δ𝑐 =1/𝐿*Δ𝑦 $$
當掉期需要的流動性比當前 bin 中可用的更多時，掉期將耗盡每個連續 bin 中的流動性，然後再轉移到下一個具有流動性的相鄰 bin。

### 4.3 Swap 費用
協議將收取費用，以補償流動性提供者發生的交易活動。總掉期費用 (fs) 將包含兩個部分，一個基本費用 (fb) 和一個可變費用 (fv)，它是瞬時價格的函數揮發性。
費率將應用於每個流動性箱中的掉期金額，並在分配給協議後按比例分配給該箱中的流動性提供者。
費用將與流動性分開並由流動性提供者索取。
可以根據公式 4.8 計算跨 n 個 bin 的掉期總費用。
![](https://i.imgur.com/P53hpt5.png)
#### 4.3.1 基本費用
基本費率 (fb) 是 bin 步長 (s) 的函數，並按基本費率 (B) 縮放，如公式 4.9 所示。
基本費用代表所有掉期的最低費率，最大值等於 bin 步長。
(4.9) 
$$ f_b = B*s $$

4.3.2 可變費用
可變費用 (fv) 旨在補償流動性提供者的瞬時波動性，並激勵流動性提供者圍繞移動價格積極管理流動性。可變費用使用公式 4.10 計算每箱 (k)。

![](https://i.imgur.com/6l8oBPk.png)

瞬時波動率在新引入的波動率累積變量 vk 中被捕獲。A 是用於衡量可變費用部分的市場參數。

#### 4.3.3 波動率累加器
波動性可以從 bin 隨時間變化的數量得出。
每個 bin 變化代表一個由 bin 步驟定義的固定價格變動。需要一個累加器來將 bin 交叉的應用擴展為超越單個交易的波動性度量。
波動率累加器 (vk) 是單個交易中的 bin 交叉及其在先前交易中的值的函數，如公式 4.11 所示。

![](https://i.imgur.com/MBomjGl.png)

累加器表示在一組先前事務中發生的 bin 更改的數量。當連續事務發生在時間上限和下限之間時，累加器將按減少因子 R 衰減。
當交易頻率高時（由 filterPeriod, tl 建立），市場被認為是波動的，累加器不會衰減。當交易頻率低（由 decayPeriod, tu 建立）時，市場被認為是穩定的並且累加器將重置為 0。
為了防止過度和持續的費用，每個市場都會使用 maxAccumulator 參數 𝑣 來限制累加器的最大值。
#### 4.3.4 協議費用
為協議保留的掉期費用部分 (fp) 將由協議費用參數 (𝜙) 控制。𝜙 是一個市場參數，將管理每個市場的費用分配。收取協議費用後，剩餘部分將是發送給流動性提供者。
![](https://i.imgur.com/mUXrITy.png)

## 5 DEX 改進
### 5.1 無常損失
可變費用為流動性提供者在各種交易動態下管理無常損失提供補償。
鑑於價格波動會導致無常損失，波動累加器是一種根據市場動態調整費用的自然機制。
此外，由於波動率累加器的時間衰減，流動性提供者有機會超越這一預期收益。
這種回報如圖 6 所示。由於累加器基於 bin 步長，因此在極不穩定的事件下由於名義價格變化的大偏差而發生跟踪錯誤。

![](https://i.imgur.com/wJrP2WE.png)

### 5.2 流動性深度
提高交易所的資本效率會增加一定數量資本的市場深度。
資本效率可以定義為提供相當於 v1 的流動性所需的資本減少。
對於 LB，通過將 v1 上的掉期對價格的影響等同於單個 bin 步驟，可以找到可比較的流動性。
由該定義得出的效率方程如下所示，效率極限結果列於圖 7 中。
這些限制代表了給定 bin 步長的每個市場的最大效率。

![](https://i.imgur.com/1V6iVhY.png)

如圖 7 所示，LB 的效率與 Uniswap v3 (v3) 相當。
可以使用均勻分佈的 LB 位置來逼近 v3 位置。
如果流動性要在 X 代幣的倉中平均分配，則 LB 和 v3 之間的掉期價值在倉位上的相對差異由方程 5.2 描述。這個方程可以繪製為一組具有不同價格範圍的倉位，離散化為不同的bin 步驟，如圖 8 所示。

![](https://i.imgur.com/E7nEavq.png)
![](https://i.imgur.com/w5tYKTs.png)
auto_awesome
Translate from: English
1,150 / 5,000
Translation results
如圖 8 所示，頭寸價格範圍是相對市場深度的重要因素，而不是倉結構。事實上，對於跨越至少 2 個倉的倉位，跨步選擇的深度差異可以忽略不計。
使用上面指定的分佈，LB 的市場深度在 Uniswap v3 的 1% 以內，對於任何跨度低於 60% 價格範圍的頭寸。此外，深度差異幾乎完全是由於跨範圍內流動性分佈的選擇（與 v3 的幾何序列相比）並且僅在大範圍位置上才有意義。
這些發現僅進一步說明了 LB 協議實現的靈活流動性分配的價值。
## 6 結論
流動性賬簿 (LB) 是一種用於構建去中心化交易所流動性的新穎設計。它允許將流動性離散到固定價格箱中，改善滑點和掉期定價。與以前的集中流動性協議不同，LB 避免了流動性提供者的高無常損失。 LB 流動性結構允許進一步的可組合性，我們熱衷於與 DeFi 社區一起探索新的用例。