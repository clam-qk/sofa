# 说明

### 介绍

sofa链是使用Cosmos SDK构建，并运行在CometBFT（Tendermint Core的一个分支）共识引擎之上，实现快速终结性、高交易吞吐量和短区块时间（大约2秒）。sofa允许用户执行Cosmos和EVM格式的交易，开发者可以通过IBC扩展EVM dApps跨链，网络中的代币和资产可以来自不同的独立来源。

### 功能

* 利用[Cosmos SDK实现的](https://docs.cosmos.network/)[模块](https://docs.cosmos.network/v0.47/build/building-modules/intro) 和其他机制。
* 实现 CometBFT 的应用程序区块链接口（[ABCI](https://docs.tendermint.com/master/spec/abci/)）来管理区块链。
* 用作[`geth`](https://github.com/ethereum/go-ethereum)库来促进代码重用并提高可维护性。
* 完全兼容的 Web3 JSON-RPC层，与现有的以太坊客户端和工具（Metamask、Remix、Truffle 等）进行交互。

这些功能的总和使得开发人员能够利用现有的以太坊生态系统工具和软件无缝部署与 Cosmos [生态系统](https://cosmos.network/ecosystem)其余部分交互的智能合约。

### 版本

* Cosmos SDK v0.50.5
* go-ethereum v1.14.5
* cometbft v0.38.6
*   Golang `>=1.21.0`

