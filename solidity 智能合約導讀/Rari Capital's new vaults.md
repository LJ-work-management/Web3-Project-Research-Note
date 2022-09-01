官網：https://rari.capital/
video: https://www.youtube.com/watch?v=yz3GygIjcUE&list=PLtQA_IktTCnZcITKc6Bj2Y8jtf33n5ZDk&index=5
code: https://github.com/Rari-Capital/vaults

## Vaults
靈活、簡約和**氣體優化的收益聚合器協議**，用於賺取任何 ERC20 代幣的利息。
-   [文檔](https://docs.rari.capital/vaults)
-   [部署](https://github.com/Rari-Capital/vaults/releases)
-   [白皮書](https://github.com/Rari-Capital/vaults/blob/main/whitepaper/Whitepaper.pdf)
-   [審計](https://github.com/Rari-Capital/vaults/blob/main/audits)
## Architecture
-   [`Vault.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/Vault.sol): 靈活、簡約和氣體優化的收益聚合器，用於賺取任何 ERC20 代幣的利息。
-   [`VaultFactory.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/VaultFactory.sol): 可以為任何 ERC20 代幣部署 Vault 合約的工廠。
-   [`modules/`](https://github.com/Rari-Capital/vaults/blob/main/src/modules)：用於管理和/或簡化與 Vault 和 Vault Factory 交互的合同。
    -   [`VaultRouterModule.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/modules/VaultRouterModule.sol): 允許通過許可存入 ETH 和免批准存款的模塊。
    -   [`VaultConfigurationModule.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/modules/VaultConfigurationModule.sol): 用於配置 Vault 參數的模塊。
    -   [`VaultInitializationModule.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/modules/VaultInitializationModule.sol): 用於初始化新創建的 Vault 的模塊。
-   [`interfaces/`](https://github.com/Rari-Capital/vaults/blob/main/src/interfaces): 外部合約 Vaults 和模塊交互的接口。
    -   [`Strategy.sol`](https://github.com/Rari-Capital/vaults/blob/main/src/interfaces/Strategy.sol)：ERC20 和 ETH 兼容策略的最小接口。

用dapp tool來管理其智能合約

![[Pasted image 20220410170658.png]]

## 影片筆記
使用js style的import
import {ERC20} from "solmate/tokens/ERC20.sol";

先看`VaultFactory.sol`
再看`Vault.sol`
### solmate
Rari他們自己的庫
https://github.com/Rari-Capital/solmate
用於智能合約開發的現代、固執且gas優化的構建塊。
跟OZ功能重疊
但solmate可能會有更好的效率,因為它不考慮向後兼容,而OZ則可能略顯擁腫