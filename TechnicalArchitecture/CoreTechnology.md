# 核心技术

### **Cosmos SDK**

sofa实现了Cosmos SDK的完全可组合性和模块化。作为一条 Cosmos 链，sofa 是一条拥有自己的原生代币的主权区块链，可以通过 IBC 连接到其他链。它包括来自 Cosmos SDK 的标准模块，这些模块与 sofa 核心开发团队构建的 sofa 特定模块协同工作。在模块列表里面可以知道每个模块负责的内容。

### CometBFT  <a href="#cometbft--abci" id="cometbft--abci"></a>

[CometBFT](https://github.com/cometbft/cometbft)由两个主要技术组件组成：区块链共识引擎和通用应用程序接口。共识引擎确保相同的交易以相同的顺序记录在每台机器上。应用程序接口称为应用程序区块链接口 (ABCI)，它允许使用任何编程语言处理交易。

CometBFT 已发展成为一种通用区块链共识引擎，可以承载任意应用程序状态。由于它可以复制任意应用程序，因此可以用作其他区块链共识引擎的即插即用替代品。Evmos 是一个通过 CometBFT 共识引擎取代以太坊 PoW 的 ABCI 应用程序示例。

另一个基于 CometBFT 构建的加密货币应用示例是 Cosmos 网络。CometBFT 可以通过在应用程序流程和共识流程之间提供一个非常简单的 API（即 ABCI）来分解区块链设计。

### EVM

Evmos 通过实现各种组件来实现 EVM 兼容性，这些组件共同支持所有 EVM 状态转换，同时确保与以太坊相同的开发人员体验：

* 以太坊的交易格式作为 Cosmos SDK`Tx`和`Msg`接口
* 以太坊的`secp256k1`Cosmos Keyring 曲线
* `StateDB`状态更新和查询接口
* 用于与 EVM 交互的JSON-RPC客户端

核心功能都在x/evm 模块中实现， 但是为了实现无缝的开发人员用户体验，有些组件是在模块之外实现的。
