# ovm

![](./img/01.png)

* **CTC（Canonical Transaction Chain）合约**：主要负责管理 Layer 2 上所有交易的顺序和记录。它维护一个交易链，将所有从 Layer 2 发送到 Layer 1 的交易聚合在一起。管理和记录了所有 Layer 2 的交易。
* **SCC（State Commitment Chain）合约**：负责管理 Layer 2 的状态根（State Root），即 Layer 2 上所有账户和智能合约的状态快照。它验证并记录这些状态更新，以确保 Layer 2 的状态与 Layer 1 的一致性。管理和验证了这些交易执行后的状态根。
* **信使合约（Message Contract）**：主要用于在不同的链之间传递消息。消息可以是状态更新、事件通知、函数调用等，这些消息不一定涉及资产的转移。
* **桥合约（Bridge Contract）**： 桥合约主要用于在不同的链之间进行资产的转移，确保跨链资产的安全性和一致性。
* **batch-submiter**: 主要用于将 L2 网络上的交易数据批量提交到 L1（Layer 1）上
* **sequencer**：`sequencer`（排序者）是一个负责管理交易顺序、打包和提交交易到主链的角色
* **proposer**：提交交易状态（stateRoot）到ETH 
* Geth：使用go编写的以太坊核心代码 

 