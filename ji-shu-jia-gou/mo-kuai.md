# 模块

{% hint style="info" %}
&#x20;模块化是cosmos sdk提供的核心功能，自定义模块化代码放在x目录下面，然后需要在app.go中注册对应的模块化实现keeper，sofa链主要有以下几个模块。
{% endhint %}

{% tabs %}
{% tab title="evm" %}
自 2015 年以太坊推出以来，通过[**智能合约**](https://www.fon.hum.uva.nl/rob/Courses/InformationInSpeech/CDROM/Literature/LOTwinterschool2006/szabo.best.vwh.net/idea.html)控制数字资产的能力 吸引了大量开发者在以太坊虚拟机 (EVM) 上构建去中心化应用程序。该社区不断创建大量工具并引入标准，这进一步提高了 EVM 兼容技术的采用率。

然而，基于 EVM 的区块链（例如以太坊）的发展也暴露出了一些可扩展性挑战，这些挑战通常被称为去[中心化、安全性和可扩展性的三难困境](https://vitalik.eth.limo/general/2021/04/07/sharding.html)。开发人员对高昂的 gas 费用、缓慢的交易速度和吞吐量以及由于部署的应用程序范围广泛而只能缓慢改变的链特定治理感到沮丧。需要一种解决方案来消除开发人员在熟悉的 EVM 环境中构建应用程序的这些担忧。

该`x/evm`模块在可扩展、高吞吐量的权益证明区块链上提供 EVM 熟悉度。它构建为[Cosmos SDK 模块](https://docs.cosmos.network/main/build/building-modules/intro) ，允许部署智能合约、与 EVM 状态机交互（状态转换）以及使用 EVM 工具。它可以在 Cosmos 应用程序专用区块链上使用，通过[Tendermint Core 的高交易吞吐量、快速交易终结性和](https://github.com/tendermint/tendermint)[IBC 的](https://ibcprotocol.org/)水平可扩展性缓解上述问题。

该模块是[ethermint 库](https://pkg.go.dev/github.com/evmos/ethermint)`x/evm`的一部分。

### 内容 <a href="#contents" id="contents"></a>

1. **Concepts**
2. **State**
3. **State Transitions**
4. **Transactions**
5. **ABCI**
6. **Hooks**
7. **Events**
8. **Parameters**
9. **Client**

### 模块架构 <a href="#module-architecture" id="module-architecture"></a>

> **注意：**如果您不熟悉 SDK 模块的整体模块结构，请检查此[文档](https://docs.cosmos.network/main/build/building-modules/structure)作为先决阅读。

```
evm/
├── client
│   └── cli
│       ├── query.go      # CLI query commands for the module
│       └── tx.go         # CLI transaction commands for the module
├── keeper
│   ├── keeper.go         # ABCI BeginBlock and EndBlock logic
│   ├── keeper.go         # Store keeper that handles the business logic of the module and has access to a specific subtree of the state tree.
│   ├── params.go         # Parameter getter and setter
│   ├── querier.go        # State query functions
│   └── statedb.go        # Functions from types/statedb with a passed in sdk.Context
├── types
│   ├── chain_config.go
│   ├── codec.go          # Type registration for encoding
│   ├── errors.go         # Module-specific errors
│   ├── events.go         # Events exposed to the Tendermint PubSub/Websocket
│   ├── genesis.go        # Genesis state for the module
│   ├── journal.go        # Ethereum Journal of state transitions
│   ├── keys.go           # Store keys and utility functions
│   ├── logs.go           # Types for persisting Ethereum tx logs on state after chain upgrades
│   ├── msg.go            # EVM module transaction messages
│   ├── params.go         # Module parameters that can be customized with governance parameter change proposals
│   ├── state_object.go   # EVM state object
│   ├── statedb.go        # Implementation of the StateDb interface
│   ├── storage.go        # Implementation of the Ethereum state storage map using arrays to prevent non-determinism
│   └── tx_data.go        # Ethereum transaction data types
├── genesis.go            # ABCI InitGenesis and ExportGenesis functionality
├── handler.go            # Message routing
└── module.go             # Module setup for the module manager
```

### **Concepts** <a href="#concepts" id="concepts"></a>

#### EVM <a href="#evm-1" id="evm-1"></a>

以太坊虚拟机 (EVM) 是一个计算引擎，可以看作是由运行以太坊客户端的数千台联网计算机（节点）维护的单一实体。作为虚拟机 ( [VM](https://en.wikipedia.org/wiki/Virtual\_machine) )，EVM 负责确定性地计算状态变化，而不管其环境（硬件和操作系统）如何。这意味着，在给定相同起始状态和交易 (tx) 的情况下，每个节点都必须获得完全相同的结果。

EVM 被认为是以太坊协议的一部分，用于处理[智能合约](https://ethereum.org/en/developers/docs/smart-contracts/)的部署和执行。为了明确区分：

* 以太坊协议描述了区块链，所有以太坊账户和智能合约都存在于其中。它在链中的任何给定区块中都只有一个规范状态（保存所有账户的数据结构）。
* 然而，EVM 是定义计算区块间新有效状态的规则的[状态机](https://en.wikipedia.org/wiki/Finite-state\_machine) 。它是一个独立的运行时，这意味着在 EVM 内部运行的代码无法访问网络、文件系统或其他进程（不是外部 API）。

该`x/evm`模块将 EVM 实现为 Cosmos SDK 模块。它允许用户通过提交以太坊 txs 并在给定状态下执行其包含的消息来与 EVM 进行交互，以引发状态转换。

状态

以太坊状态是一种数据结构，以[Merkle Patricia 树](https://en.wikipedia.org/wiki/Merkle\_tree)的形式实现，将所有账户保存在链上。EVM 会对此数据结构进行更改，从而产生具有不同状态根的新状态。因此，以太坊可以看作是一条状态链，它通过使用 EVM 在区块中执行交易来从一个状态转换到另一个状态。新的交易区块可以通过其区块头（父哈希、区块编号、时间戳、随机数、收据等）进行描述。

**账户**

有两种类型的账户可以在给定地址中存储状态：

* **外部拥有账户（EOA）**：具有 nonce（tx 计数器）和余额
* **智能合约**：具有随机数、余额、（不可变的）代码哈希、存储根（另一个 Merkle Patricia Trie）

智能合约就像区块链上的常规账户一样，它还以以太坊特定的二进制格式存储可执行代码，称为**EVM 字节码**。它们通常用以太坊高级语言编写，例如 Solidity，它被编译为 EVM 字节码并通过使用以太坊客户端提交交易来部署在区块链上。

**结构**

EVM 以堆栈为基础运行。其主要架构组件包括：

* 虚拟 ROM：处理交易时，合约代码被拉入此只读存储器中
* 机器状态（易失性）：随着 EVM 的运行而变化，并在处理每个 tx 后被清除
  * 程序计数器（PC）
  * Gas：跟踪使用了多少 gas
  * 堆栈和内存：计算状态变化
* 访问账户存储（持久性）

**智能合约**

通常，智能合约会公开公共 ABI，即用户与合约交互所支持的一系列方式。要与合约交互并调用状态转换，用户将提交一个交易，其中包含任意数量的 gas 和根据 ABI 格式化的数据有效负载，指定交互类型和任何其他参数。收到交易后，EVM 将使用交易有效负载执行智能合约的 EVM 字节码。

**执行 EVM字节码**

合约的 EVM 字节码由基本操作（加、乘、存储等）组成，这些操作称为**操作码**。每个操作码的执行都需要随交易支付的 gas。因此，EVM 被认为是准图灵完备的，因为它允许任意计算，但是合约执行期间的计算量限制为交易中提供的 gas 量。每个操作码的[**gas 成本**](https://www.evm.codes/)反映了在实际计算机硬件上运行这些操作的成本（例如`ADD = 3gas`和`SSTORE = 100gas`）。要计算交易的 gas 消耗，需要将 gas 成本乘以 gas**价格**，该价格会根据当时网络的需求而变化。如果网络负载过重，您可能需要支付更高的 gas 价格才能执行您的交易。如果达到 gas 限制（gas 耗尽例外），则不会对以太坊状态进行任何更改，但发送者的 nonce 会增加，其余额会减少，以支付浪费 EVM 时间的代价。

智能合约还可以调用其他智能合约。每次调用新合约都会创建一个新的 EVM 实例（包括新的堆栈和内存）。每次调用都会将沙盒状态传递给下一个 EVM。如果 gas 耗尽，则所有状态更改都将被丢弃。否则，它们将被保留。

如需进一步阅读，请参阅：

* [以太坊虚拟机](https://eth.wiki/concepts/evm/evm)
* [EVM 架构](https://cypherpunks-core.github.io/ethereumbook/13evm.html#evm\_architecture)
* [什么是以太坊](https://ethdocs.org/en/latest/introduction/what-is-ethereum.html#what-is-ethereum)
* [操作码](https://www.ethervm.io/)

#### Sofa 作为 Geth 的实现 <a href="#evmos-as-geth-implementation" id="evmos-as-geth-implementation"></a>

Sofa 包含以太坊协议在 Golang (Geth) 中的实现， 作为 Cosmos SDK 模块。Geth 包含 EVM 的实现，用于计算状态转换。查看[go-ethereum 源代码](https://github.com/ethereum/go-ethereum/blob/master/core/vm/instructions.go) ，了解 EVM 操作码的实现方式。正如 Geth 可以作为以太坊节点运行一样，Sofa 可以作为节点运行，使用 EVM 计算状态转换。Sofa 支持 Geth 的标准以太坊 JSON-RPC API， 以便与 Web3 和 EVM 兼容。

**JSON- RPC**

JSON-RPC 是一种无状态的轻量级远程过程调用 (RPC) 协议。该规范主要定义了几种数据结构及其处理规则。它与传输无关，因为这些概念可以在同一进程内、通过套接字、通过 HTTP 或在许多不同的消息传递环境中使用。它使用 JSON (RFC 4627) 作为数据格式。

**JSON-RPC 示例`eth_call`**[**：**](https://docs.evmos.org/protocol/modules/evm#json-rpc-example-eth\_call)

JSON-RPC 方法`eth_call`允许您针对合约执行消息。通常，您需要将交易发送到 Geth 节点以将其包含在内存池中，然后节点之间相互传播，最终将交易包含在块中并执行。 `eth_call`允许您将数据发送到合约并查看在不提交交易的情况下发生的情况。

在Geth实现中，调用端点大致经过以下步骤：

1. 请求`eth_call`被转换为`func (s *PublicBlockchainAPI) Call()`使用`eth`命名空间调用函数
2. [`Call()`](https://github.com/ethereum/go-ethereum/blob/master/internal/ethapi/api.go#L982)被赋予交易参数、要调用的区块以及修改要调用的状态的可选参数。然后它调用`DoCall()`。
3. [`DoCall()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/internal/ethapi/api.go#L891) 将参数转换为`ethtypes.message`，实例化 EVM 并使用`core.ApplyMessage`
4. [`ApplyMessage()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state\_transition.go#L180) 调用状态转换`TransitionDb()`
5. [`TransitionDb()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state\_transition.go#L275) 要么`Create()`签订新合同，要么`Call()`签订 SA 合同
6. [`evm.Call()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/evm.go#L168) 运行解释器`evm.interpreter.Run()`执行消息。如果执行失败，则状态恢复为执行前的快照并消耗 gas。
7. [`Run()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/interpreter.go#L116) 执行循环来执行操作码。

Sofa 的实现类似，并利用了 Cosmos SDK 中包含的 gRPC 查询客户端：

1. `eth_call`请求被转换为`func (e *PublicAPI) Call`使用`eth`命名空间调用函数
2. `Call()`调用`doCall()`
3. `doCall()` 将参数转换为`EthCallRequest`并`EthCall()`使用 evm 模块的查询客户端进行调用。
4. `EthCall()` 将参数转换为`ethtypes.message`并调用“ApplyMessageWithConfig()
5. `ApplyMessageWithConfig()`使用 Geth 实现 实例化 EVM 和`Create()`sa 新合约或sa 合约。`Call()`

**StateDB**

[go-ethereum](https://github.com/ethereum/go-ethereum/blob/master/core/vm/interface.go)`StateDB`的接口代表 用于完整状态查询的 EVM 数据库。EVM 状态转换由此接口启用，该接口在模块中由 实现。此接口的实现使 Sofa EVM 兼容。`x/evmKeeper`

#### 共识引擎 <a href="#consensus-engine" id="consensus-engine"></a>

使用该模块的应用程序`x/evm`通过应用程序区块链接口 (ABCI) 与 Tendermint Core 共识引擎进行交互。应用程序和 Tendermint Core 共同构成运行完整区块链的程序，并将业务逻辑与去中心化数据存储相结合。

提交给`x/evm`模块的以太坊交易在执行并更改应用程序状态之前会参与此共识过程。我们鼓励您了解[Tendermint 共识引擎](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)的基础知识 ，以便详细了解状态转换。

#### 交易日志 <a href="#transaction-logs" id="transaction-logs"></a>

在每笔`x/evm`交易中，结果都包含`Log`来自状态机执行的以太坊，JSON-RPC Web3 服务器使用这些以太坊进行过滤查询和处理 EVM Hooks。

交易日志在交易执行期间存储在临时存储中，然后在交易处理完成后通过 cosmos 事件发出。可以通过 gRPC 和 JSON-RPC 查询它们。

#### StateBlock Bloom <a href="#block-bloom" id="block-bloom"></a>

Bloom 是每个区块的布隆过滤器值（以字节为单位），可用于过滤器查询。区块布隆过滤器值存储在临时存储中，然后在`EndBlock`处理过程中通过 cosmos 事件发出。它们可以通过 gRPC 和 JSON-RPC 进行查询。

提示

👉**注意**：由于它们不存储在状态中，因此升级后交易日志和区块布隆不会保留。升级后，用户必须使用存档节点才能获取遗留链事件。

### State <a href="#state-1" id="state-1"></a>

本节概述了`x/evm`模块状态中存储的对象、从 go-ethereum 接口派生的功能`StateDB`、通过 Keeper 的实现以及genesis的状态实现。

#### 状态对象 <a href="#state-objects" id="state-objects"></a>

该`x/evm`模块保持以下对象的状态：

State

|             | 描述                                                               | Key                           | Value               | Store     |
| ----------- | ---------------------------------------------------------------- | ----------------------------- | ------------------- | --------- |
| Code        | 智能合约字节码                                                          | `[]byte{1} + []byte(address)` | `[]byte{code}`      | KV        |
| Storage     | 智能合约存储                                                           | `[]byte{2} + [32]byte{key}`   | `[32]byte(value)`   | KV        |
| Block Bloom | 区块布隆过滤器，用于累积当前区块的布隆过滤器，在结束区块时发射到事件。                              | `[]byte{1} + []byte(tx.Hash)` | `protobuf([]Log)`   | Transient |
| Tx Index    | 当前块中当前交易的索引。                                                     | `[]byte{2}`                   | `BigEndian(uint64)` | Transient |
| Log Size    | 当前区块中迄今为止发出的日志数量。用于决定后续日志的日志索引。                                  | `[]byte{3}`                   | `BigEndian(uint64)` | Transient |
| Gas Used    | 当前 cosmos-sdk tx 的以太坊消息所使用的 gas 数量，当 cosmos-sdk tx 包含多个以太坊消息时需要。 | `[]byte{4}`                   | `BigEndian(uint64)` | Transient |

#### StateDB <a href="#statedb-1" id="statedb-1"></a>

该接口由模块中的`StateDB`实现，用于表示用于查询合约和账户完整状态的 EVM 数据库。在以太坊协议中，用于存储 IAVL 树中的任何内容，并负责缓存和存储嵌套状态。`StateDBx/evm/statedbStateDB`

```
// github.com/ethereum/go-ethereum/core/vm/interface.go
type StateDB interface {
 CreateAccount(common.Address)

 SubBalance(common.Address, *big.Int)
 AddBalance(common.Address, *big.Int)
 GetBalance(common.Address) *big.Int

 GetNonce(common.Address) uint64
 SetNonce(common.Address, uint64)

 GetCodeHash(common.Address) common.Hash
 GetCode(common.Address) []byte
 SetCode(common.Address, []byte)
 GetCodeSize(common.Address) int

 AddRefund(uint64)
 SubRefund(uint64)
 GetRefund() uint64

 GetCommittedState(common.Address, common.Hash) common.Hash
 GetState(common.Address, common.Hash) common.Hash
 SetState(common.Address, common.Hash, common.Hash)

 Suicide(common.Address) bool
 HasSuicided(common.Address) bool

 // Exist reports whether the given account exists in state.
 // Notably this should also return true for suicided accounts.
 Exist(common.Address) bool
 // Empty returns whether the given account is empty. Empty
 // is defined according to EIP161 (balance = nonce = code = 0).
 Empty(common.Address) bool

 PrepareAccessList(sender common.Address, dest *common.Address, precompiles []common.Address, txAccesses types.AccessList)
 AddressInAccessList(addr common.Address) bool
 SlotInAccessList(addr common.Address, slot common.Hash) (addressOk bool, slotOk bool)
 // AddAddressToAccessList adds the given address to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddAddressToAccessList(addr common.Address)
 // AddSlotToAccessList adds the given (address,slot) to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddSlotToAccessList(addr common.Address, slot common.Hash)

 RevertToSnapshot(int)
 Snapshot() int

 AddLog(*types.Log)
 AddPreimage(common.Hash, []byte)

 ForEachStorage(common.Address, func(common.Hash, common.Hash) bool) error
}
```

`StateDB`中的提供`x/evm`了以下功能：

**以太坊账户**

您可以`EthAccount`从提供的地址创建实例，并将值设置为存储在 `AccountKeeper`with上`createAccount()`。 如果具有给定地址的帐户已存在，则此函数还会重置与该地址关联的任何预先存在的代码和存储。

可以通过 来管理帐户的硬币余额，`BankKeeper` 并且可以使用 来读取`GetBalance()`，使用`AddBalance()`和来更新`SubBalance()`。

* `GetBalance()`返回指定地址的EVM面额余额，面额从模块参数中获取。
* `AddBalance()`通过铸造新币并将其转移到地址，将给定金额添加到地址余额币中。 币面额从模块参数中获取。
* `SubBalance()`通过将代币转移到托管账户然后销毁，从地址余额中减去给定金额。代币面额从模块参数中获取。如果金额为负数或用户没有足够的资金进行转账，则此函数不执行任何操作。

`Sequence`可以通过 auth 模块从 Account 中获取 nonce（或者说交易序列）`AccountKeeper`。

* `GetNonce()`检索指定地址的账户并返回 tx 序列（即 nonce）。如果未找到该账户，该函数将执行无操作。
* `SetNonce()`将给定的 nonce 设置为地址账户的序列。如果账户不存在，则从地址创建一个新账户。

包含任意合约逻辑的智能合约字节码存储在 上`EVMKeeper` ，可以使用`GetCodeHash()`、`GetCode()`&进行查询`GetCodeSize()`，并使用 进行更新`SetCode()`。

* `GetCodeHash()`从存储中获取帐户并返回其代码哈希。如果帐户不存在或不是 EthAccount 类型，则返回空的代码哈希值。
* `GetCode()`返回与给定地址关联的代码字节数组。如果帐户的代码哈希为空，则此函数返回 nil。
* `SetCode()`将代码字节数组存储到应用程序 KVStore，并将代码哈希设置为给定帐户。如果代码为空，则从存储中删除该代码。
* `GetCodeSize()`返回与此对象关联的合约代码的大小，如果没有，则返回零。

需要跟踪并存储已退还的 Gas，以便在 EVM 执行完成后将其添加到已使用 Gas 值中或从中减去/添加。每次交易和每个区块结束时都会清除退款值。

* `AddRefund()`将给定量的 gas 添加到内存中的退款值中。
* `SubRefund()`从内存中的退款值中减去给定的 gas 量。如果 gas 量大于当前退款，则此函数将引发恐慌。
* `GetRefund()`返回交易执行完成后可返还的 gas 数量。每次交易后，此值都会重置为 0。

状态存储在 上`EVMKeeper`。可以使用 进行查询`GetCommittedState()`，`GetState()`并使用 进行更新`SetState()`。

* `GetCommittedState()`返回为给定的键哈希设置的存储值。如果键未注册，此函数将返回空哈希。
* `GetState()`返回给定键哈希的内存中脏状态，如果不存在，则从 KVStore 加载提交的值。
* `SetState()`将给定的哈希值（key，value）设置为状态。如果值哈希为空，则此函数从状态中删除键，新值最初保持为脏状态，最终将提交给 KVStore。

账户也可以设置为自杀状态。当合约自杀时，账户被标记为自杀，提交代码时，存储和账户将被删除（从下一个区块开始）。

* `Suicide()`将给定帐户标记为已自杀，并清除 EVM 代币的帐户余额。
* `HasSuicided()`查询内存中的标志，检查账户是否在当前交易中被标记为自杀。自杀的账户将在查询期间返回为非零，并在提交区块后被“清除”。

要检查帐户存在，请使用`Exist()`和`Empty()`。

* `Exist()`如果给定的帐户存在于商店中或者已被标记为自杀，则返回 true。
* `Empty()`如果地址满足以下条件则返回 true：
  * nonce 为 0
  * evm denom 的余额为 0
  * 帐户代码哈希为空

**EIP2930功能**

[支持包含访问列表](https://eips.ethereum.org/EIPS/eip-2930)（交易计划访问的地址和存储密钥列表）的交易类型。访问列表状态保存在内存中，并在交易提交后丢弃。

* `PrepareAccessList()`处理针对 EIP-2929 和 EIP-2930 执行状态转换的准备步骤。仅当 Yolov3/Berlin/2929+2930 适用于当前编号时才应调用此方法。
  * 将发件人添加到访问列表（EIP-2929）
  * 将目的地添加到访问列表（EIP-2929）
  * 将预编译添加到访问列表（EIP-2929）
  * 添加可选 tx 访问列表的内容（EIP-2930）
* `AddressInAccessList()`如果地址已注册，则返回 true。
* `SlotInAccessList()`检查地址和插槽是否已注册。
* `AddAddressToAccessList()`将给定的地址添加到访问列表中。如果该地址已经在访问列表中，则此函数不执行任何操作。
* `AddSlotToAccessList()`将给定的 (address, slot) 添加到访问列表。如果地址和 slot 已经在访问列表中，则此函数不执行任何操作。

**快照状态和恢复功能**

EVM 使用状态恢复异常来处理错误。此类异常将撤消对当前调用（及其所有子调用）中的状态所做的所有更改，并且调用者可以处理错误并且不会传播。您可以使用 来`Snapshot()`识别修订版中的当前状态，并使用 将状态恢复到给定的修订版以`RevertToSnapshot()`支持此功能。

* `Snapshot()`创建一个新的快照并返回标识符。
* `RevertToSnapshot(rev)`撤消对标识为 的快照的所有修改`rev`。

Sofa 改编了[go-ethereum 日志实现](https://github.com/ethereum/go-ethereum/blob/master/core/state/journal.go#L39) 来支持这一点，它使用日志列表来记录迄今为止完成的所有状态修改操作，快照由唯一的 id 和日志列表中的索引组成，并且为了恢复到快照，它只是以相反的顺序撤消快照索引后的日志。

**以太坊交易日志**

您`AddLog()`可以将给定的以太坊附加`Log`到与当前状态中保存的交易哈希相关联的日志列表中。此函数还会在设置要存储的日志之前填写 tx 哈希、块哈希、tx 索引和日志索引字段。

#### Keeper <a href="#keeper" id="keeper"></a>

EVM 模块`Keeper`授予对 EVM 模块状态的访问权限，并实现`statedb.Keeper`接口以支持实现`StateDB`。Keeper 包含一个存储密钥，允许 DB 写入多存储的具体子树，该子树只能由 EVM 模块访问。Sofa 不使用 trie 和数据库进行查询和持久化（实现`StateDB`），而是使用 Cosmos `KVStore`（键值存储）和 Cosmos SDK`Keeper`来促进状态转换。

为了支持接口功能，它导入了 4 个模块 Keepers：

* `auth`: CRUD 帐户
* `bank`：会计（供应）和余额的 CRUD
* `staking`：查询历史头信息
* `fee marketDynamicFeeTx` ：EIP-1559 硬分叉激活后处理基础费用`London`已在`ChainConfig`参数上

```
type Keeper struct {
 // Protobuf codec
 cdc codec.BinaryCodec
 // Store key required for the EVM Prefix KVStore. It is required by:
 // - storing account's Storage State
 // - storing account's Code
 // - storing Bloom filters by block height. Needed for the Web3 API.
 // For the full list, check the module specification
 storeKey sdk.StoreKey

 // key to access the transient store, which is reset on every block during Commit
 transientKey sdk.StoreKey

 // module specific parameter space that can be configured through governance
 paramSpace paramtypes.Subspace
 // access to account state
 accountKeeper types.AccountKeeper
 // update balance and accounting operations with coins
 bankKeeper types.BankKeeper
 // access historical headers for EVM state transition execution
 stakingKeeper types.StakingKeeper
 // fetch EIP1559 base fee and parameters
 feeMarketKeeper types.FeeMarketKeeper

 // chain ID number obtained from the context's chain id
 eip155ChainID *big.Int

 // Tracer used to collect execution traces from the EVM transaction execution
 tracer string
 // trace EVM state transition execution. This value is obtained from the `--trace` flag.
 // For more info check https://geth.ethereum.org/docs/dapp/tracing
 debug bool

 // EVM Hooks for tx post-processing
 hooks types.EvmHooks
}
```

#### Genesis State <a href="#genesis-state" id="genesis-state"></a>

该`x/evm`模块`GenesisState`定义了从先前导出的高度初始化链所需的状态。它包含`GenesisAccounts`和模块参数

```
type GenesisState struct {
  // accounts is an array containing the ethereum genesis accounts.
  Accounts []GenesisAccount `protobuf:"bytes,1,rep,name=accounts,proto3" json:"accounts"`
  // params defines all the parameters of the module.
  Params Params `protobuf:"bytes,2,opt,name=params,proto3" json:"params"`
}
```

#### Genesis Accounts <a href="#genesis-accounts" id="genesis-accounts"></a>

该`GenesisAccount`类型对应于以太坊类型的改编`GenesisAccount`。它定义了在创世状态下要初始化的账户。

其主要区别在于，Sofa 上的使用自定义`Storage`类型，该类型使用切片而不是 evm 的映射`State`（由于不确定性），并且它不包含私钥字段。

还需要注意的是，由于`auth`Cosmos SDK 上的模块管理帐户状态，因此该字段必须与 存储在模块中的`Address`现有字段相对应（即）。地址使用 上的[**EIP55**](https://eips.ethereum.org/EIPS/eip-55)十六进制[**格式**](https://docs.evmos.org/protocol/concepts/accounts#address-formats-for-clients)。`EthAccountauthKeeperAccountKeepergenesis.json`

```
type GenesisAccount struct {
  // address defines an ethereum hex formated address of an account
  Address string `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
  // code defines the hex bytes of the account code.
  Code string `protobuf:"bytes,2,opt,name=code,proto3" json:"code,omitempty"`
  // storage defines the set of state key values for the account.
  Storage Storage `protobuf:"bytes,3,rep,name=storage,proto3,castrepeated=Storage" json:"storage"`
}
```

### **State Transitions** <a href="#state-transitions" id="state-transitions"></a>

该`x/evm`模块允许用户提交以太坊交易（`Tx`）并执行其包含的消息以在给定状态下引起状态转换。

用户在客户端提交交易以将其广播到网络。当交易在共识期间包含在区块中时，它将在服务器端执行。我们强烈建议您了解[Tendermint 共识引擎](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)的基础知识 ，以详细了解状态转换。

#### 客户端 <a href="#client-side" id="client-side"></a>

提示

👉 这是基于`eth_sendTransaction`JSON-RPC 的

1. 用户使用与以太坊兼容的客户端或钱包（例如 Metamask、WalletConnect、Ledger 等）通过可用的 JSON-RPC 端点之一提交交易：a. eth（公共）命名空间：
   * `eth_sendTransaction`
   * `eth_sendRawTransaction` b.个人（私有）命名空间：
   * `personal_sendTransaction`
2. `MsgEthereumTx`在填充 RPC 交易后，将创建一个实例，使用`SetTxDefaults`默认值填充缺失的交易参数
3. `Tx`使用以下方式验证字段（无状态）`ValidateBasic()`
4. 使用与发件人地址相关联的密钥和最新的以太坊硬分叉（、等）进行`Tx`签名**。**`LondonBerlinChainConfig`
5. 使用 Cosmos Config 构建器根据`Tx`msg**字段**构建
6. `Tx`以[同步模式](https://docs.cosmos.network/main/user/run-node/txs#broadcasting-a-transaction)广播以 确保等待执行响应。交易**由**应用程序使用进行验证，然后添加到共识引擎的内存池中。[`CheckTx`](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)`CheckTx()`
7. JSON-RPC 用户收到包含交易字段哈希值的响应。此哈希值与 SDK Transactions 使用的计算交易字节哈希值的[`RLP`](https://eth.wiki/en/fundamentals/rlp)默认哈希值不同。`sha256`

#### 服务器端 <a href="#server-side" id="server-side"></a>

一旦在共识期间提交了一个块（包含`Tx`），它就会在服务器端的一系列 ABCI 消息中应用于应用程序。

`Tx`应用程序通过调用 来处理每个事务。在对 中 [`RunTx`](https://docs.cosmos.network/main/learn/advanced/baseapp)的每个事务进行无状态验证后 ，确认 是以太坊事务还是 SDK 事务。作为以太坊事务，它包含的消息随后由模块处理以更新应用程序的状态。`sdk.MsgTxAnteHandlerTxx/evm`

**AnteHandler**

`anteHandler`每笔交易都会运行。它会检查是否是`Tx`以太坊交易，并将其路由到内部 ante 处理程序。在这里，`Tx`使用 EthereumTx 扩展选项处理 ，以不同于普通 Cosmos SDK 交易的方式处理它们。为每个`antehandler`运行一系列选项及其`AnteHandle`功能`Tx`：

* `EthSetUpContextDecorator()`改编自 cosmos-sdk 中的 SetUpContextDecorator，它通过将 gas 表设置为无限来忽略 gas 消耗
* `EthValidateBasicDecorator(evmKeeper)Tx`验证以太坊类型 Cosmos消息的字段
* `EthSigVerificationDecorator(evmKeeper)`验证注册的链 ID 与消息上的 ID 是否相同，以及签名者地址是否与消息上定义的地址匹配。RecheckTx 不会跳过此步骤，因为它设置了`From`其他 ante 处理程序必须使用的地址。RecheckTx 失败将阻止 tx 被纳入区块，尤其是当 CheckTx 成功时，在这种情况下用户不会看到错误消息。
* `EthAccountVerificationDecorator(ak, bankKeeper, evmKeeper)` 将验证发送方余额是否大于总交易成本。如果账户不存在，即无法在商店中找到，则将账户设置为商店。如果出现以下情况，此 AnteHandler 装饰器将失败：
  * 任何消息都不是 MsgEthereumTx
  * 发件人地址为空
  * 账户余额低于交易成本
* `EthNonceVerificationDecorator(ak)`验证交易 nonce 是否有效，且等同于发送方账户的当前 nonce。
* `EthGasConsumeDecorator(evmKeeper)`验证以太坊 tx 消息是否有足够的 gas（仅在 CheckTx 期间）以及发送者是否有足够的余额来支付 gas 成本。交易的内在 gas 是交易执行前使用的 gas 量。gas 是一个常数值加上交易提供的额外数据字节所产生的任何成本。如果出现以下情况，此 AnteHandler 装饰器将失败：
  * 交易包含多条消息
  * 该消息不是 MsgEthereumTx
  * 找不到发件人账户
  * 交易的 gas 限制低于固有 gas
  * 用户没有足够的余额来扣除交易费用（gas\_limit \* gas\_price）
  * 交易或区块 gas 表耗尽 gas
* `CanTransferDecorator(evmKeeper, feeMarketKeeper)`从消息中创建一个EVM，并调用BlockContext CanTransfer函数来查看该地址是否可以执行交易。
* `EthIncrementSenderSequenceDecorator(ak)` 处理签名者（即发送者）序列的递增。如果交易是合约创建，则 nonce 将在交易执行期间递增，而不是在此 AnteHandler 装饰器内递增。

选项`authante.NewMempoolFeeDecorator()`、`authante.NewTxTimeoutHeightDecorator()` 和`authante.NewValidateMemoDecorator(ak)`与 Cosmos 相同`Tx`。单击 [此处](https://docs.cosmos.network/main/learn/beginner/gas-fees.html#antehandler) 获取有关 的更多信息 `anteHandler`。

**EVM模块**

通过 身份验证后`antehandler`， 中的每个`sdk.Msg`（在本例中`MsgEthereumTx`）`Tx` 被传送到模块中的 Msg Handler`x/evm`并执行以下步骤：

1. 转换`Msg`为以太坊`Tx`类型
2. 申请`Tx`并`EVMConfig`尝试执行状态转换，如果事务没有失败，则仅将状态转换持久化（提交）到底层 KVStore：
   1. 确认已`EVMConfig`创建
   2. 使用链配置值创建以太坊签名者`EVMConfig`
   3. 将以太坊交易哈希设置为（非永久）临时存储，以便它也可以在 StateDB 函数中使用
   4. 生成新的EVM实例
   5. `EnableCreate`确认合约创建（ ）和合约执行（ ）的 EVM 参数`EnableCall`已启用
   6. 应用消息。如果`To`地址为`nil`，则使用代码作为部署代码创建新合约。否则，使用给定的输入作为参数调用给定地址的合约
   7. 计算 evm 操作使用的 gas
3. 如果`Tx`申请成功
   1. 执行 EVM`Tx`后处理钩子。如果钩子返回错误，则恢复整个`Tx`
   2. 根据以太坊 gas 会计规则退还 gas
   3. 使用 tx 生成的日志更新区块布隆过滤器的值
   4. 为交易字段和交易日志发出 SDK 事件

### **Transitions** <a href="#transactions" id="transactions"></a>

本节定义 `sdk.Msg` 导致上一节定义的状态转换的具体类型。

### `MsgEthereumTx`[​](https://docs.evmos.org/protocol/modules/evm#msgethereumtx) <a href="#msgethereumtx" id="msgethereumtx"></a>

可以使用 实现 EVM 状态转换`MsgEthereumTx`。此消息将以太坊交易数据（`TxData`）封装为`sdk.Msg`。它包含必要的交易数据字段。请注意， 实现`MsgEthereumTx`了[`sdk.Msg`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx\_msg.go#L7-L29) 和[`sdk.Tx`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx\_msg.go#L33-L41)接口。通常，SDK 消息仅实现前者，而后者是一组捆绑在一起的消息。

```
type MsgEthereumTx struct {
 // inner transaction data
 Data *types.Any `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
 // DEPRECATED: encoded storage size of the transaction
 Size_ float64 `protobuf:"fixed64,2,opt,name=size,proto3" json:"-"`
 // transaction hash in hex format
 Hash string `protobuf:"bytes,3,opt,name=hash,proto3" json:"hash,omitempty" rlp:"-"`
 // ethereum signer address in hex format. This address value is checked
 // against the address derived from the signature (V, R, S) using the
 // secp256k1 elliptic curve
 From string `protobuf:"bytes,4,opt,name=from,proto3" json:"from,omitempty"`
}
```

如果发生以下情况，此消息字段验证预计会失败：

* `From`字段已定义且地址无效
* `TxData`无状态验证失败

如果发生以下情况，交易执行可能会失败：

* 任何自定义`AnteHandler`以太坊装饰器检查失败：
  * 交易的最低gas量要求
  * 发送方账户不存在或余额不足以支付费用
  * 账户顺序与交易不匹配`Data.AccountNonce`
  * 消息签名验证失败
* EVM 合约创建（即`evm.Create`）失败，或者`evm.Call`失败

**Conversion**

可以`MsgEthreumTx`转换为 go-ethereum`Transaction`和`Message`类型以创建和调用 evm 合约。

```
// AsTransaction creates an Ethereum Transaction type from the msg fields
func (msg MsgEthereumTx) AsTransaction() *ethtypes.Transaction {
 txData, err := UnpackTxData(msg.Data)
 if err != nil {
  return nil
 }

 return ethtypes.NewTx(txData.AsEthereumData())
}

// AsMessage returns the transaction as a core.Message.
func (tx *Transaction) AsMessage(s Signer, baseFee *big.Int) (Message, error) {
 msg := Message{
  nonce:      tx.Nonce(),
  gasLimit:   tx.Gas(),
  gasPrice:   new(big.Int).Set(tx.GasPrice()),
  gasFeeCap:  new(big.Int).Set(tx.GasFeeCap()),
  gasTipCap:  new(big.Int).Set(tx.GasTipCap()),
  to:         tx.To(),
  amount:     tx.Value(),
  data:       tx.Data(),
  accessList: tx.AccessList(),
  isFake:     false,
 }
 // If baseFee provided, set gasPrice to effectiveGasPrice.
 if baseFee != nil {
  msg.gasPrice = math.BigMin(msg.gasPrice.Add(msg.gasTipCap, baseFee), msg.gasFeeCap)
 }
 var err error
 msg.from, err = Sender(s, tx)
 return msg, err
}
```

**签名**

为了使签名验证有效，必须 `TxData`包含`v | r | s`来自的值`Signer`。Sign 计算 secp256k1 ECDSA 签名并签署交易。它采用密钥环签名者和 chainID 来根据 EIP-155 标准签署以太坊交易。此方法在填充交易签名的 V、R、S 字段时会改变交易。如果未为消息定义发送方地址或未在密钥环上注册发送方，则该函数将失败。

```
// Sign calculates a secp256k1 ECDSA signature and signs the transaction. It
// takes a keyring signer and the chainID to sign an Ethereum transaction according to
// EIP-155 standard.
// This method mutates the transaction as it populates the V, R, S
// fields of the Transaction's Signature.
// The function will fail if the sender address is not defined for the msg or if
// the sender is not registered on the keyring
func (msg *MsgEthereumTx) Sign(ethSigner ethtypes.Signer, keyringSigner keyring.Signer) error {
 from := msg.GetFrom()
 if from.Empty() {
  return fmt.Errorf("sender address not defined for message")
 }

 tx := msg.AsTransaction()
 txHash := ethSigner.Hash(tx)

 sig, _, err := keyringSigner.SignByAddress(from, txHash.Bytes())
 if err != nil {
  return err
 }

 tx, err = tx.WithSignature(ethSigner, sig)
 if err != nil {
  return err
 }

 msg.FromEthereumTx(tx)
 return nil
}
```

#### 交易数据 <a href="#txdata" id="txdata"></a>

支持`MsgEthereumTx`来自 go-ethereum 的 3 种有效以太坊交易数据类型： `LegacyTx`、`AccessListTx` 和`DynamicFeeTx`。这些类型被定义为 protobuf 消息并打包到字段`proto.Any`中的接口类型中`MsgEthereumTx`。

* `LegacyTx`：[EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)交易类型
* `DynamicFeeTx`：[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)交易类型。由伦敦硬分叉区块启用
* `AccessListTx`：[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)交易类型。由柏林硬分叉区块启用

#### `LegacyTx`[​](https://docs.evmos.org/protocol/modules/evm#legacytx) <a href="#legacytx" id="legacytx"></a>

以太坊常规交易的交易数据。

```
type LegacyTx struct {
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,1,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,3,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,4,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,5,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data []byte `protobuf:"bytes,6,opt,name=data,proto3" json:"data,omitempty"`
 // v defines the signature value
 V []byte `protobuf:"bytes,7,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,8,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,9,opt,name=s,proto3" json:"s,omitempty"`
}
```

如果发生以下情况，此消息字段验证预计会失败：

* `GasPrice`无效（`nil`、负数或者超出 int256 范围）
* `Fee`(gasprice \*gaslimit) 无效
* `Amount`无效（负数或超出 int256 范围）
* `To`地址无效（非有效的以太坊十六进制地址）

#### `DynamicFeeTx`[​](https://docs.evmos.org/protocol/modules/evm#dynamicfeetx) <a href="#dynamicfeetx" id="dynamicfeetx"></a>

EIP-1559 动态费用交易的交易数据。

```
type DynamicFeeTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas tip cap defines the max value for the gas tip
 GasTipCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_tip_cap,json=gasTipCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_tip_cap,omitempty"`
 // gas fee cap defines the max value for the gas fee
 GasFeeCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=gas_fee_cap,json=gasFeeCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_fee_cap,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,5,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,6,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,7,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,8,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,9,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,10,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,11,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,12,opt,name=s,proto3" json:"s,omitempty"`
}
```

如果发生以下情况，此消息字段验证预计会失败：

* `GasTipCap`无效（`nil`、负数或溢出 int256）
* `GasFeeCap`无效（`nil`、负数或溢出 int256）
* `GasFeeCap`小于`GasTipCap`
* `Fee`(gas price \* gas limit) 无效（溢出 int256）
* `Amount`无效（负数或溢出 int256）
* `To`地址无效（非有效的以太坊十六进制地址）
* `ChainID`是`nil`

#### `AccessListTx`[​](https://docs.evmos.org/protocol/modules/evm#accesslisttx) <a href="#accesslisttx" id="accesslisttx"></a>

EIP-2930 访问列表交易的交易数据。

```
type AccessListTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,4,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,5,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,6,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,7,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,8,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,9,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,10,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,11,opt,name=s,proto3" json:"s,omitempty"`
}
```

如果发生以下情况，此消息字段验证预计会失败：

* `GasPrice`无效（`nil`、负数或溢出 int256）
* `Fee`(gas price \* gas limit) 无效（溢出 int256）
* `Amount`无效（负数或溢出 int256）
* `To`地址无效（非有效的以太坊十六进制地址）
* `ChainID`是`nil`

### ABCI <a href="#abci" id="abci"></a>

应用程序区块链接口 (ABCI) 允许应用程序与 Tendermint 共识引擎进行交互。应用程序与 Tendermint 维护多个 ABCI 连接。与 最相关的 `x/evm`是[Commit 处的共识连接](https://docs.tendermint.com/v0.33/app-dev/app-development.html#consensus-connection)。此连接 负责块执行并调用函数 `InitChain` （包含`InitGenesis`）  `BeginBlock`、、、、  。  仅在第一次启动新区块链时调用，并 为块中的 每个  交易调用。`DeliverTxEndBlockCommitInitChainDeliverTx`

#### InitGenesis <a href="#initgenesis" id="initgenesis"></a>

`InitGenesis`通过设置存储字段来初始化 EVM 模块创世状态`GenesisState`。具体来说，它设置参数和创世账户（状态和代码）。

#### ExportGenesis <a href="#exportgenesis" id="exportgenesis"></a>

ABCI`ExportGenesis`函数导出 EVM 模块的创世状态。具体来说，它会检索所有帐户及其字节码、余额和存储、交易日志以及 EVM 参数和链配置。

#### BeginBlock <a href="#beginblock" id="beginblock"></a>

EVM 模块`BeginBlock`逻辑在处理交易的状态转换之前执行。此功能的主要目的是：

* 设置当前块的上下文，以便在 EVM 状态转换期间调用`Keeper`其中一个函数时，可以使用块头、存储、gas 计等。`StateDB`
* 设置 EIP-155`ChainID`编号（从完整链 ID 获得），以防在之前没有设置过`InitChain`

#### EndBlock <a href="#endblock" id="endblock"></a>

EVM 模块`EndBlock`逻辑在执行交易的所有状态转换后发生。此功能的主要目的是：

* 发出 Block bloom 事件
  * 这是为了兼容 web3，因为以太坊标头包含此类型作为字段。JSON-RPC 服务使用此事件查询从 Tendermint 标头构建以太坊标头。
  * 从临时存储中获取块布隆过滤器值，然后将其发出

### Hooks <a href="#hooks" id="hooks"></a>

模块`x/evm`实现了对外`EvmHooks`扩展和定制处理逻辑的接口。`Tx`

这支持 EVM 合约通过以下方式调用原生的 cosmos 模块

1. 定义日志签名并从智能合约发出特定日志，
2. 在原生 tx 处理代码中识别这些日志，并且
3. 将它们转换为本机模块调用。

为此，接口包含一个 `PostTxProcessing`钩子，用于`Tx`在中注册自定义钩子`EvmKeeper`。这些 `Tx`钩子在 EVM 状态转换完成且不会失败后进行处理。请注意，EVM 模块中没有实现默认钩子。

```
type EvmHooks interface {
 // Must be called after tx is processed successfully, if return an error, the whole transaction is reverted.
 PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error
}
```

### `PostTxProcessing`[​](https://docs.evmos.org/protocol/modules/evm#posttxprocessing) <a href="#posttxprocessing" id="posttxprocessing"></a>

`PostTxProcessing`仅在 EVM 交易成功完成后调用，并将调用委托给底层钩子。如果未注册钩子，此函数将返回错误`nil`。

```
func (k *Keeper) PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error {
 if k.hooks == nil {
  return nil
 }
 return k.hooks.PostTxProcessing(k.Ctx(), msg, receipt)
}
```

它在与 EVM 交易相同的缓存上下文中执行，如果它返回错误，则整个 EVM 交易将被还原，如果钩子实现者不想还原 tx，他们总是可以返回`nil`。

钩子返回的错误被翻译成虚拟机错误`failed to process native logs`，详细的错误信息存储在返回值中。该消息被异步发送到本机模块，调用者无法捕获和恢复错误。

### Events <a href="#events" id="events"></a>

该`x/evm`模块在状态执行后发出 Cosmos SDK 事件。EVM 模块发出相关交易字段的事件以及交易日志（以太坊事件）。

#### MsgEthereumTx <a href="#msgethereumtx-1" id="msgethereumtx-1"></a>

| 类型           | 属性键                | 属性值                     |
| ------------ | ------------------ | ----------------------- |
| ethereum\_tx | `"amount"`         | `{amount}`              |
| ethereum\_tx | `"recipient"`      | `{hex_address}`         |
| ethereum\_tx | `"contract"`       | `{hex_address}`         |
| ethereum\_tx | `"txHash"`         | `{tendermint_hex_hash}` |
| ethereum\_tx | `"ethereumTxHash"` | `{hex_hash}`            |
| ethereum\_tx | `"txIndex"`        | `{tx_index}`            |
| ethereum\_tx | `"txGasUsed"`      | `{gas_used}`            |
| tx\_log      | `"txLog"`          | `{tx_log}`              |
| message      | `"sender"`         | `{eth_address}`         |
| message      | `"action"`         | `"ethereum"`            |
| message      | `"module"`         | `"evm"`                 |

此外，EVM 模块在`EndBlock`过滤器查询block bloom期间发出一个事件。

#### ABCI <a href="#abci-1" id="abci-1"></a>

| 类型           | 属性键       | 属性值                  |
| ------------ | --------- | -------------------- |
| block\_bloom | `"bloom"` | `string(bloomBytes)` |

### Parameters <a href="#parameters" id="parameters"></a>

evm模块包含以下参数：

#### 参数 <a href="#params" id="params"></a>

| Key                   | 类型            | 默认值                |
| --------------------- | ------------- | ------------------ |
| `EVMDenom`            | string        | `"asofa"`          |
| ~~`EnableCreate`~~    | bool          | `true`             |
| ~~`EnableCall`~~      | bool          | `true`             |
| `ExtraEIPs`           | \[]int        | TBD                |
| `ChainConfig`         | ChainConfig   | see ChainConfig    |
| `AllowUnprotectedTxs` | bool          | false              |
| `ActivePrecompiles`   | \[]string     | \[]                |
| `AccessControl`       | AccessControl | Permissionless EVM |

#### EVM名称 <a href="#evm-denom" id="evm-denom"></a>

evm 面额参数定义了 EVM 状态转换中使用的代币面额和 EVM 消息的 gas 消耗。

例如，在以太坊上， 将`evm_denom`是`ETH`。在 Sofa 的情况下，默认面额是sofa。在精度方面，SOFA和`ETH`共享相同的值， _即_ `1 SOFA = 10^18 asofa`和`1 ETH = 10^18 wei`。

提示

注意：想要导入 EVM 模块作为依赖项的 SDK 应用程序需要设置自己的模块`evm_denom`（即不`"asofa"`）。

#### 启用创建 <a href="#enable-create" id="enable-create"></a>

&#x20;启用创建参数可切换使用该`vm.Create`函数的状态转换。禁用该参数时，它将阻止所有合约创建功能。

#### 启用Transfer <a href="#enable-transfer" id="enable-transfer"></a>

**（在 v19.0.0 中已弃用）** 启用转账可切换使用该`vm.Call`函数的状态转换。当禁用该参数时，它将阻止账户之间的转账和执行智能合约调用。

#### 额外的EIP <a href="#extra-eips" id="extra-eips"></a>

额外的 EIPs 参数定义了在以太坊虚拟机上应用自定义跳转表的一组可激活的以太坊改进提案（ [**EIP ）。**](https://ethereum.org/en/eips/)`Config`

提示

注意：其中一些 EIP 已通过链配置启用，具体取决于硬分叉号。

支持的可激活 EIPS 包括：

* [**EIP 1344**](https://eips.ethereum.org/EIPS/eip-1344)
* [**EIP 1884**](https://eips.ethereum.org/EIPS/eip-1884)
* [**EIP 2200**](https://eips.ethereum.org/EIPS/eip-2200)
* [**EIP 2315**](https://eips.ethereum.org/EIPS/eip-2315)
* [**EIP 2929**](https://eips.ethereum.org/EIPS/eip-2929)
* [**EIP 3198**](https://eips.ethereum.org/EIPS/eip-3198)
* [**EIP 3529**](https://eips.ethereum.org/EIPS/eip-3529)
* [**EIP 3855**](https://eips.ethereum.org/EIPS/eip-3855)

#### Chain Config <a href="#chain-config" id="chain-config"></a>

是`ChainConfig`一个 protobuf 包装器类型，它包含与 go-ethereum 参数相同的字段`ChainConfig`，但使用`*sdk.Int`类型而不是`*big.Int`。

默认情况下，除 之外的所有块配置字段`ConstantinopleBlock`都在创世块（高度 0）处启用。

**ChainConfig默认值**

| Name                | 默认值                                                                  |
| ------------------- | -------------------------------------------------------------------- |
| HomesteadBlock      | 0                                                                    |
| DAOForkBlock        | 0                                                                    |
| DAOForkSupport      | `true`                                                               |
| EIP150Block         | 0                                                                    |
| EIP150Hash          | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| EIP155Block         | 0                                                                    |
| EIP158Block         | 0                                                                    |
| ByzantiumBlock      | 0                                                                    |
| ConstantinopleBlock | 0                                                                    |
| PetersburgBlock     | 0                                                                    |
| IstanbulBlock       | 0                                                                    |
| MuirGlacierBlock    | 0                                                                    |
| BerlinBlock         | 0                                                                    |
| LondonBlock         | 0                                                                    |
| ArrowGlacierBlock   | 0                                                                    |
| GrayGlacierBlock    | 0                                                                    |
| MergeNetsplitBlock  | 0                                                                    |
| ShanghaiBlock       | 0                                                                    |
| CancunBlock         | 0                                                                    |

#### 允许不受保护的交易 <a href="#allow-unprotected-transactions" id="allow-unprotected-transactions"></a>

此参数会为参与共识的所有节点全局强制执行EIP-155 重放保护。如果禁用，则会将重放保护的控制权委托给各个节点。

#### 主动预编译 <a href="#active-precompiles" id="active-precompiles"></a>

此参数控制在给定网络上启用哪些EVM 扩展 。它接受十六进制格式的地址列表，该列表在 EVM 交易中进行评估，以仅允许与选定的预编译合约进行交互。

#### 访问控制 <a href="#access-control" id="access-control"></a>

此参数可对 EVM 进行详细控制。先前的参数`enable_create`已`enable_call`扩展，可精确控制谁可以访问这些功能。

默认情况下，EVM 是_无需许可的_，这意味着任何人都可以部署智能合约并发送 EVM 交易，除非他们被特别列入黑名单。黑名单地址可以在相应的 中定义`AccessControlList`。

`AccessControlType`通过将创建或调用功能设置为_受限_，EVM 不接受与特定功能的进一步交互。

当将控制类型定义为_许可_时，给定的地址列表将被解释为白名单地址的集合，这些地址是唯一能够部署合约或调用 EVM 的地址。

### Client <a href="#client" id="client"></a>

用户可以 `evm` 使用 CLI、JSON-RPC、gRPC 或 REST 查询并与模块交互。

#### CLI <a href="#cli" id="cli"></a>

`sofad` 以下是随模块添加的命令 列表 `x/evm` 。您可以使用该命令获取完整列表 sofa`d -h` 。

**查询**

这些 `query` 命令允许用户查询 `evm` 状态。

**`code`**

允许用户查询给定地址的智能合约代码。

```
sofad query evm code ADDRESS [flags]
```

```
# Example
$ sofad query evm code 0x7bf7b17da59880d9bcca24915679668db75f9397

# Output
code: "0xef616c92f3cfc9e92dc270d6acff9cea213cecc7020a76ee4395af09bdceb4837a1ebdb5735e11e7d3adb6104e0c3ac55180b4ddf5e54d022cc5e8837f6a4f971b"
```

**`storage`**

允许用户使用给定的密钥和高度查询帐户的存储。

```
sofad query evm storage ADDRESS KEY [flags]
```

```
# Example
$ sofad query evm storage 0x0f54f47bf9b8e317b214ccd6a7c3e38b893cd7f0 0 --height 0

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

**Transactions**

这些 `tx` 命令允许用户与模块交互 `evm` 。

**`raw`**

允许用户从原始以太坊交易构建宇宙交易。

```
sofad tx evm raw TX_HEX [flags]
```

```
# Example
$ sofad tx evm raw 0xf9ff74c86aefeb5f6019d77280bbb44fb695b4d45cfe97e6eed7acd62905f4a85034d5c68ed25a2e7a8eeb9baf1b84

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

\

{% endtab %}

{% tab title="feemarket" %}
本文档指定了费用市场模块，该模块允许为网络定义全球交易费用。

该模块旨在支持 cosmos-sdk 中的 EIP-1559。

需要覆盖模块以检查`MempoolFeeDecorator`并允许实施根据网络活动而变化的全局费用机制。`x/authbaseFeeminimal-gas-prices`

有关 EIP-1559 的更多参考：

[https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)

### 内容 <a href="#contents" id="contents"></a>

1. **Concepts**
2. **State**
3. **Begin Block**
4. **End Block**
5. **Keeper**
6. **Events**
7. **Params**
8. **Client**
9. **AntHandlers**

### **Concepts** <a href="#concepts" id="concepts"></a>

#### EIP-1559：Fee Market <a href="#eip-1559-fee-market" id="eip-1559-fee-market"></a>

[EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)描述了以太坊提出的一种定价机制，旨在改进交易费用的计算。它包括每区块固定的网络费用，该费用将被销毁，并动态扩展/收缩区块大小以应对网络拥塞高峰。

在 EIP-1559 之前，交易费用的计算方式如下

```
fee = gasPrice * gasLimit
```

其中`gasPrice`是每 gas 的价格，`gasLimit`描述了执行交易所需的 gas 量。交易所需的操作越复杂，gas 限制就越高（请参阅[执行 EVM 字节码](https://docs.evmos.org/protocol/modules/evm#executing-evm-bytecode)）。要提交交易，签名者需要指定`gasPrice`。

启用 EIP-1559 后，交易费用的计算方式如下：

```
fee = (baseFee + priorityTip) * gasLimit
```

其中`baseFee`是固定的每区块网络费用（每 gas），`priorityTip`是可选设置的每 gas 额外费用。请注意，基本费用和优先小费都是 gas 价格。要使用 EIP-1559 提交交易，签名者需要指定`gasFeeCap`，这是他们愿意支付的每 gas 最高费用。也`priorityTip`可以指定 ，它涵盖优先费用和区块的每 gas 网络费用（又名：基本费用）。

提示

Cosmos SDK 使用的术语与`gas`以太坊不同。以太坊上的名称与 Cosmos 上的`gasLimit`名称相同`gasWanted`。您可能会在 Sofa 上遇到这两种术语，因为它在 SDK 之上构建了以太坊，例如在使用不同的钱包时，例如 Cosmos 的 Keplr 和以太坊的 Metamask。

#### Base Fee <a href="#base-fee" id="base-fee"></a>

每 Gas 基础费用（又称基础费用）是在共识层面定义的全局 Gas 价格。它作为模块参数存储，并根据前一个块中使用的 Gas 总量和 Gas 目标（`block gas limit / elasticity multiplier`）在每个块的末尾进行调整：

* 当方块高于气体目标时，它会增加，
* 当块低于气体目标时，它会减少。

该模块不会销毁基础费用（如在以太坊上实施的那样），而是`feemarket`将基础费用分配给常规[Cosmos SDK 费用分配](https://docs.cosmos.network/main/modules/distribution)。

#### Priority Tip <a href="#priority-tip" id="priority-tip"></a>

在 EIP-1559 中，`max_priority_fee_per_gas`通常称为`tip`，是可以添加到 的额外 gas 价格，`baseFee`用于激励交易优先排序。小费越高，交易被纳入区块的可能性就越大。

然而，直到 Cosmos SDK 版本 v0.46 之前，没有交易优先级的概念。因此，Sofa 上 EIP-1559 交易的小费应该为零（`MaxPriorityFeePerGas`JSON-RPC 端点返回`0`）。查看内存池文档以了解有关如何利用交易优先级的更多信息。

#### Effective Gas price <a href="#effective-gas-price" id="effective-gas-price"></a>

对于 EIP-1559 交易（动态费用交易），有效 gas 价格描述了交易愿意提供的最高 gas 价格。它来自交易参数和基本费用参数。根据两者中较小者，有效 gas 价格为`baseFee + tip`或`gasFeeCap`

```
min(baseFee + gasTipCap, gasFeeCap)
```

#### Local vs. Global Minimum Gas Prices <a href="#local-vs-global-minimum-gas-prices" id="local-vs-global-minimum-gas-prices"></a>

最低 gas 价格用于丢弃网络中的垃圾交易，方法是将交易成本提高到垃圾交易者无法承受的程度。这是通过为 Cosmos 和 EVM 交易定义内存池中接受交易的最低 gas 价格来实现的。如果交易不提供以下两种最低 gas 价格中的至少一种，则会从内存池中丢弃该交易：

最低 gas 价格用于丢弃网络中的垃圾交易，方法是将交易成本提高到垃圾交易者无法承受的程度。这是通过为 Cosmos 和 EVM 交易定义内存池中接受交易的最低 gas 价格来实现的。如果交易不提供以下两种最低 gas 价格中的至少一种，则会从内存池中丢弃该交易：

1. 验证者可以在其节点配置上设置的本地最低 gas 价格以及
2. 全局最低 gas 价格，该价格在模块中设置为参数`feemarket`，可以通过治理进行更改。

交易 gas 价格的下限是通过比较以下三种情况的 gas 价格界限来确定的：

1. 如果有效 gas 价格 ( `effective gas price = base fee + priority tip`) 或局部最低 gas 价格低于全局`MinGasPrice` ( `min-gas-price (local) < MinGasPrice (global) OR EffectiveGasPrice < MinGasPrice`)，则将`MinGasPrice`其用作下限。
2. 如果交易因 gas 价格低于 而被拒绝`MinGasPrice`，则用户需要重新发送 gas 价格高于或等于 的交易`MinGasPrice`。
3. 如果有效 gas 价格或本地 gas 价格`minimum-gas-price`高于全局 gas`MinGasPrice`价格，则以两者中较大的值作为下限。在 EIP-1559 的情况下，用户必须提高优先费才能使其交易有效。

交易 gas price 与下限的比较是通过 AnteHandler 装饰器实现的。对于 EVM 交易，这是在 和 中完成的`EthMempoolFeeDecorator`；`EthMinGasPriceDecorator` `AnteHandler` 对于 Cosmos 交易，这是在`NewMempoolFeeDecorator`和中完成的`MinGasPriceDecorator` `AnteHandler`。

提示

如果基础费用下降到低于全局 的值`MinGasPrice`，则将其设置为`MinGasPrice`。这样实施的目的是，基础费用不会因为 而下降到无法在内存池中接受交易的 gas 价格`MinGasPrice`。

### State <a href="#state" id="state"></a>

x/feemarket 模块保存费用计算所需的状态变量：

只需要跟踪前一个区块中的 BlockGasUsed 即可进行下一次基本费用计算。

|              | 描述                    | Key         | Value              | Store |
| ------------ | --------------------- | ----------- | ------------------ | ----- |
| BlockGasUsed | gas used in the block | `[]byte{1}` | `[]byte{gas_used}` | KV    |

### Begin Block <a href="#begin-block" id="begin-block"></a>

基本费用在每个区块开始时计算。

#### Base Fee <a href="#base-fee-1" id="base-fee-1"></a>

**Disabling base fee**

我们引入两个参数：`NoBaseFee`和`EnableHeight`

`NoBaseFee`控制费用市场基础费用值。如果设置为 true，则不进行任何计算，并且看管者返回的基础费用为零。

`EnableHeight`控制我们开始计算的高度。

* 如果`NoBaseFee = false`和`height < EnableHeight`，则基本费用值将等于`base_fee`创世中定义的 ，并且`BeginBlock`将返回而无需进一步计算。
* 如果`NoBaseFee = false`和`height >= EnableHeight`，则基本费用将在每个区块上动态计算`BeginBlock`。

这些参数允许我们引入静态基本费用或在稍后阶段激活基本费用。

**Enabling base fee**

要在 EVM 中启用 EIP-1559，应设置以下参数：

* NoBaseFee 应该是 false
* EnableHeight 应设置为 >= 升级高度的正整数。它定义链在哪个高度开始基本费用调整
* LondonBlock evm 的参数应设置为 >= 升级高度的正整数。它定义了链在哪个高度开始接受 EIP-1559 交易。

**Calculation**

基本费用初始化为`EnableHeight`创世`InitialBaseFee`文件中定义的值。

基本费用将根据前一个区块中使用的 Gas 总量进行调整。

```
parent_gas_target = parent_gas_limit / ELASTICITY_MULTIPLIER

if EnableHeight == block.number
    base_fee = INITIAL_BASE_FEE
else if parent_gas_used == parent_gas_target:
    base_fee = parent_base_fee
else if parent_gas_used > parent_gas_target:
    gas_used_delta = parent_gas_used - parent_gas_target
    base_fee_delta = max(parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR, 1)
    base_fee = parent_base_fee + base_fee_delta
else:
    gas_used_delta = parent_gas_target - parent_gas_used
    base_fee_delta = parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR
    base_fee = parent_base_fee - base_fee_delta

```

### End Block <a href="#end-block" id="end-block"></a>

该`block_gas_used`值在每个块的末尾更新。

#### Block Gas Used <a href="#block-gas-used" id="block-gas-used"></a>

当前块使用的总 gas 存储在 KVStore 中`EndBlock`。

它被初始化为`block_gas`在创世纪中定义的。

### Keeper <a href="#keeper" id="keeper"></a>

feemarket 模块提供了这个导出的 keeper，可以传递给需要访问基本费用值的其他模块

```
type Keeper interface {
    GetBaseFee(ctx sdk.Context) *big.Int
}
```

### Events <a href="#events" id="events"></a>

该`x/feemarket`模块发出以下事件：

#### BeginBlocker <a href="#beginblocker" id="beginblocker"></a>

| Type        | Attribute Key | Attribute Value |
| ----------- | ------------- | --------------- |
| fee\_market | base\_fee     | {baseGasPrices} |

#### EndBlocker <a href="#endblocker" id="endblocker"></a>

| Type       | Attribute Key | Attribute Value |
| ---------- | ------------- | --------------- |
| block\_gas | height        | {blockHeight}   |
| block\_gas | amount        | {blockGasUsed}  |

### Parameters <a href="#parameters" id="parameters"></a>

该`x/feemarket`模块包含以下参数：

| Key                      | Type    | 默认值        | 描述                               |
| ------------------------ | ------- | ---------- | -------------------------------- |
| NoBaseFee                | bool    | false      | 控制基本费用调整                         |
| BaseFeeChangeDenominator | int32   | 8          | 限制区块之间可以改变的基本费用金额                |
| ElasticityMultiplier     | int32   | 2          | 根据前一个区块使用的总 gas 来限制基本费用的增加或减少的阈值 |
| BaseFee                  | int32   | 1000000000 | EIP-1559 区块的基本费用                 |
| EnableHeight             | int32   | 0          | 可以调整费用的高度                        |
| MinGasPrice              | sdk.Dec | 0          | 将交易打包到区块中需要支付的全局最低 gas 价格        |

### Client <a href="#client" id="client"></a>

#### CLI <a href="#cli" id="cli"></a>

用户可以 `feemarket` 使用 CLI 查询并与模块交互。

**Queries**

这些 `query` 命令允许用户查询 `feemarket` 状态。

```
sofad query feemarket --help
```

**Base Fee**

该`base-fee`命令允许用户按高度查询区块基础费用。

```
sofad query feemarket base-fee [flags]
```

例子：

```
sofad query feemarket base-fee ...
```

示例输出：

```
base_fee: "512908936"
```

**Block Gas**

该`block-gas`命令允许用户按高度查询区块 gas。

```
sofad query feemarket block-gas [flags]
```

例子：

```
sofad query feemarket block-gas ...
```

示例输出：

```
gas: "21000"
```

**Params**

该`params`命令允许用户查询模块参数。

```
sofad query params subspace [subspace] [key] [flags]
```

例子：

```
sofad query params subspace feemarket ElasticityMultiplier ...
```

示例输出：

```
key: ElasticityMultiplier
subspace: feemarket
value: "2"
```

#### gRPC <a href="#grpc" id="grpc"></a>

**Queries**

| Verb   | 方法                                      | 描述          |
| ------ | --------------------------------------- | ----------- |
| `gRPC` | `ethermint.feemarket.v1.Query/Params`   | 获取模块参数      |
| `gRPC` | `ethermint.feemarket.v1.Query/BaseFee`  | 获取区块基础费用    |
| `gRPC` | `ethermint.feemarket.v1.Query/BlockGas` | 获取区块使用的 gas |
| `GET`  | `/ethermint/feemarket/v1/params`        | 获取模块参数      |
| `GET`  | `/ethermint/feemarket/v1/base_fee`      | 获取区块基础费用    |
| `GET`  | `/ethermint/feemarket/v1/block_gas`     | 获取区块使用的 gas |

### AnteHandlers <a href="#antehandlers" id="antehandlers"></a>

该`x/feemarket`模块提供`AnteDecorator`以递归方式链接在一起形成单个的[`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md)。这些装饰器对以太坊或 Cosmos SDK 交易执行基本的有效性检查，以便可以将其从交易 Mempool 中抛出。

请注意，每笔交易都会运行 ，并在和`AnteHandler`上调用。`CheckTxDeliverTx`

#### Decorators <a href="#decorators" id="decorators"></a>

#### `MinGasPriceDecorator`[​](https://docs.evmos.org/protocol/modules/feemarket#mingaspricedecorator) <a href="#mingaspricedecorator" id="mingaspricedecorator"></a>

拒绝交易费用低于 的 Cosmos SDK 交易`MinGasPrice * GasLimit`。

#### `EthMinGasPriceDecorator`[​](https://docs.evmos.org/protocol/modules/feemarket#ethmingaspricedecorator) <a href="#ethmingaspricedecorator" id="ethmingaspricedecorator"></a>

拒绝交易费用低于的 EVM 交易`MinGasPrice * GasLimit`。

* 对于`LegacyTx`和`AccessListTx`，`GasPrice * GasLimit`使用 。
* 对于 EIP-1559（_又名_ `DynamicFeeTx`），`EffectivePrice * GasLimit`使用了。

提示

**注意**：对于动态交易，如果`feemarket`公式得出的结果是`BaseFee`低于`EffectivePrice < MinGasPrices`，则用户必须增加`GasTipCap`（优先费）直到`EffectivePrice > MinGasPrices`。 的交易`MinGasPrices * GasLimit < transaction fee < EffectiveFee` 将被 拒绝`feemarket` `AnteHandle`。

#### `EthGasConsumeDecorator`[​](https://docs.evmos.org/protocol/modules/feemarket#ethgasconsumedecorator) <a href="#ethgasconsumedecorator" id="ethgasconsumedecorator"></a>

根据 EIP-1559 规范计算需要扣除的有效费用和 tx 优先级，然后在响应中扣除费用并设置 tx 优先级。

```
effectivePrice = min(baseFee + tipFeeCap, gasFeeCap)
effectiveTipFee = effectivePrice - baseFee
priority = effectiveTipFee / DefaultPriorityReduction
```

当事务中有多条消息时，选择其中优先级最低的消息。
{% endtab %}

{% tab title="vesting" %}
`x/vesting`本文档详细介绍了Sofa的内部 模块。

该`x/vesting`模块引入了`ClawbackVestingAccount`，这是一种实现 Cosmos SDK[`VestingAccount`](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types) 接口的新 vesting 账户类型。此账户用于分配受 vesting、锁定和追回约束的代币。

允许`ClawbackVestingAccount`任何两方就未来的奖励计划达成一致，其中代币随着时间的推移被授予权限。双方可以使用此帐户来执行法律合同或承诺共同的长期利益。

在这一承诺中，归属是逐步获得转移和委托已分配代币的权限的机制。此外，锁定提供了一种机制，以防止从账户转移已分配代币和执行以太坊交易的权利。归属和锁定都是在创建账户时在时间表中定义的。在任何时候，出资人都`ClawbackVestingAccount`可以执行追回以取回未归属的代币。应在合同（例如智能合约）中商定应在何种情况下执行追回。

对于 Sofa 来说，`ClawbackVestingAccount`用于向核心团队成员和顾问分配代币，以激励长期参与项目。

### 内容 <a href="#contents" id="contents"></a>

1. **Concepts**
2. **State**
3. **State Transitions**
4. **Transactions**
5. **AntHandlers**
6. **Events**
7. **Clent**

### 参考文献 <a href="#references" id="references"></a>

* SDK 归属规范：[https://docs.cosmos.network/main/modules/auth/vesting](https://docs.cosmos.network/main/modules/auth/vesting)
* SDK 归属实现：[https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting)
* Agoric 的 Vesting Clawback 账户：[https://github.com/Agoric/agoric-sdk/issues/4085](https://github.com/Agoric/agoric-sdk/issues/4085)
* Agoric 的`vestcalc`工具：[https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc)

### **Concepts** <a href="#concepts" id="concepts"></a>

#### Vesting <a href="#vesting-1" id="vesting-1"></a>

`unvested`归属是指在不转移代币所有权的情况下将其转换为代币的过程`vested`。在未归属状态下，代币不能转移到其他账户、委托给验证者或用于治理。归属计划描述了代币归属的数量和时间。第一批代币归属的持续时间称为`cliff`。

#### Lockup <a href="#lockup" id="lockup"></a>

锁定描述了代币从一种状态转换为`locked`另一种`unlocked`状态的时间表。只要所有代币都被锁定，账户就无法执行任何花费 SOFA 的交易。但是，该账户可以执行不花费 SOFA 代币的交易。此外，锁定的代币不能转移到其他账户。在代币同时被锁定和归属的情况下，可以将它们委托给验证者，但不能将它们转移到其他账户。

下表总结了受归属和锁定组合约束的代币允许执行的操作：

| Token Status           | Transfer | Delegate | Vote | Eth Txs that spend SOFA\*\* | Eth Txs that don't spend SOFA (amount = 0)\*\* |
| ---------------------- | :------: | :------: | :--: | :-------------------------: | :--------------------------------------------: |
| `locked`&`unvested`    |     ❌    |     ❌    |   ❌  |              ❌              |                        ✅                       |
| `locked`&`vested`      |     ❌    |     ✅    |   ✅  |              ❌              |                        ✅                       |
| `unlocked`&`unvested`  |     ❌    |     ❌    |   ❌  |              ❌              |                        ✅                       |
| `unlocked`& `vested`\* |     ✅    |     ✅    |   ✅  |              ✅              |                        ✅                       |

\*Staking rewards are unlocked and vested

\* \* EVM 交易仅当涉及发送锁定或未归属的 SOFA 代币时才会失败，例如将 SOFA 发送到 EOA 或智能合约（如果金额 > 0 则失败）。

#### Schedules <a href="#schedules" id="schedules"></a>

归属和锁定计划指定了代币归属或解锁的数量和时间。它们被定义为[`periods`](https://docs.cosmos.network/main/modules/auth/vesting#period) 每个期限都有自己的长度和金额。例如，典型的归属计划将从一年期开始定义，以代表归属悬崖，然后是几个月的归属期，直到分配的总归属金额归属。

使用 Agoric 的工具可以轻松创建归属或锁定计划[`vestcalc`](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc)。例如，要计算从 2022 年 1 月开始的四年归属计划（其中有一年的悬崖期），您可以运行 vestcalc：

```
vestcalc --write --start=2022-01-01 --coins=200000000000000000000000asofa --months=48 --cliffs=2023-01-01
```

#### Clawback <a href="#clawback" id="clawback"></a>

如果`ClawbackVestingAccount`违反了 的基本承诺或合同，追回机制提供了一种返还未归属资金的机制。授权执行追回的账户在创建账户时定义`ClawbackVestingAccount`。它可以是：

* 治理模块（如果允许）
* 指定为的地址`FunderAddress`

需要注意的是，账户是否启用治理回扣的信息并不与账户本身一起存储，而是直接存储在归属模块中。

当发起追回或由出资人或治理发起时，未归属的代币将发送到追回消息中指定的目标地址。如果没有指定目标地址，则默认将代币退还给出资人。

### State <a href="#state" id="state"></a>

#### State Objects <a href="#state-objects" id="state-objects"></a>

该`x/vesting`模块不会将对象保存在自己的存储中。相反，它使用 SDK`auth`模块通过[帐户接口](https://docs.cosmos.network/main/modules/auth#account-interface)将帐户对象存储在状态中。帐户作为接口对外公开，​​在内部作为追回归属帐户存储。

#### ClawbackVestingAccount <a href="#clawbackvestingaccount" id="clawbackvestingaccount"></a>

[实现Vesting Account](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types)接口的实例。它提供了一个帐户，可以保存受锁定或归属约束的贡献，但受未归属代币的收回约束，或两者兼而有之（代币归属，但仍处于锁定状态）。

```
type ClawbackVestingAccount struct {
    // base_vesting_account implements the VestingAccount interface. It contains
    // all the necessary fields needed for any vesting account implementation
    *types.BaseVestingAccount `protobuf:"bytes,1,opt,name=base_vesting_account,json=baseVestingAccount,proto3,embedded=base_vesting_account" json:"base_vesting_account,omitempty"`
    // funder_address specifies the account which can perform clawback
    FunderAddress string `protobuf:"bytes,2,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
    // start_time defines the time at which the vesting period begins
    StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
    // lockup_periods defines the unlocking schedule relative to the start_time
    LockupPeriods []types.Period `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3" json:"lockup_periods"`
    // vesting_periods defines the vesting schedule relative to the start_time
    VestingPeriods []types.Period `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3" json:"vesting_periods"`
}
```

**BaseVestingAccount**

实现`VestingAccount`接口。它包含任何归属账户实现所需的所有必要字段。

**FunderAddress**

指定提供原始代币并可执行追回的账户。

**StartTime**

定义归属和锁定计划开始的时间。

**LockupPeriods**

定义相对于开始时间的解锁计划。

**VestingPeriods**

定义相对于开始时间的归属计划。

#### Genesis State <a href="#genesis-state" id="genesis-state"></a>

该`x/vesting`模块允许在创世时定义`ClawbackVestingAccounts`。在这种情况下，账户余额必须记录在 SDK`bank`模块余额中或通过 CLI 命令自动调整`add-genesis-account`。

### State Transitions <a href="#state-transitions" id="state-transitions"></a>

该`x/vesting`模块允许状态转换，即创建和更新追回归属账户`CreateClawbackVestingAccount` 或执行未归属资金的追回`Clawback`。

#### Create Clawback Vesting Account <a href="#create-clawback-vesting-account" id="create-clawback-vesting-account"></a>

外部拥有的账户可由所有者转换为追回归属账户。创建后，所有者会指定一名出资人，该出资人能够为账户提供归属和/或锁定计划的资金。该账户还可以指定是否可以从治理中追回已归属的代币。

1. `MsgCreateClawbackVestingAccount`所有者通过其中一个客户端提交。
2. 检查是否
   1. 归属账户地址未被屏蔽。
   2. 归属账户地址处的账户还不是归属账户。
3. 在目标地址创建一个回扣归属账户，其中包含空的归属和锁定计划。

#### Fund Clawback Vesting Account <a href="#fund-clawback-vesting-account" id="fund-clawback-vesting-account"></a>

追回归属账户的出资人可以为其提供归属和/或锁定计划。如果归属账户已有资金，则计划将合并在一起。

1. `MsgFundVestingAccount`资助者通过其中一位客户提交。
2. 检查是否
   1. 归属地址不是阻止地址。
   2. 归属地址是一个追回归属账户。
   3. 至少提供一个归属或锁定时间表。如果其中一个缺失，则默认为立即归属或解锁时间表。
3. 锁定期和归属期总额相等。
4. 更新追回归属账户并将资金从资助者发送到归属账户，将任何现有计划与新资金合并。

#### Clawback <a href="#clawback-1" id="clawback-1"></a>

资金地址是唯一可以执行追回的地址。

1. `MsgClawback`资助者通过其中一位客户提交。
2. 检查是否
   1. 给出目标地址，如果没有，则默认为资助者地址
   2. 目标地址未被阻止
   3. 该账户存在且为追回归属账户
   4. 账户资金提供者与消息中的相同
3. 将未归属的代币从追回归属账户转移到目标地址，更新锁定时间表并删除未来的归属事件。

#### Update Clawback Vesting Account Funder <a href="#update-clawback-vesting-account-funder" id="update-clawback-vesting-account-funder"></a>

现有回扣归属账户的资金地址只能由当前资助者更新。

1. `MsgUpdateVestingFunder`资助者通过其中一位客户提交。
2. 检查是否
   1. 新的资助者地址未被屏蔽
   2. 归属账户存在，并且是追回归属账户
   3. 账户资金提供者与消息中的相同
3. 使用新的资助者地址更新归属账户资助者。

#### Convert Vesting Account <a href="#convert-vesting-account" id="convert-vesting-account"></a>

一旦所有代币都被归属，归属账户就可以转换回`EthAccount`。

1. `MsgConvertVestingAccount`归属账户的所有者通过其中一个客户端提交。
2. 检查是否
   1. 归属账户存在，并且是追回归属账户
   2. 归属账户的归属和锁定计划已结束
3. 将归属账户转换为`EthAccount`

### Transactions <a href="#transactions" id="transactions"></a>

本节定义`sdk.Msg`导致上一节定义的状态转换的具体类型。

#### `CreateClawbackVestingAccount`[​](https://docs.evmos.org/protocol/modules/vesting#createclawbackvestingaccount) <a href="#createclawbackvestingaccount" id="createclawbackvestingaccount"></a>

```
type MsgCreateClawbackVestingAccount struct {
    // funder_address specifies the account that will be able to fund the vesting account
    FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
    // vesting_address specifies the address that will receive the vesting tokens
    VestingAddress string `protobuf:"bytes,2,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
    // enable_gov_clawback specifies whether the governance module can clawback this account
    EnableGovClawback bool `protobuf:"varint,3,opt,name=enable_gov_clawback,json=enableGovClawback,proto3" json:"enable_gov_clawback,omitempty"`
}
```

如果出现以下情况，则消息内容无状态验证失败：

* `FunderAddress`或`VestingAddress`无效

#### `FundVestingAccount`[​](https://docs.evmos.org/protocol/modules/vesting#fundvestingaccount) <a href="#fundvestingaccount" id="fundvestingaccount"></a>

```
type MsgFundVestingAccount struct {
    // funder_address specifies the account that funds the vesting account
    FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
    // vesting_address specifies the account that receives the funds
    VestingAddress string `protobuf:"bytes,2,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
    // start_time defines the time at which the vesting period begins
    StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
    // lockup_periods defines the unlocking schedule relative to the start_time
    LockupPeriods github_com_cosmos_cosmos_sdk_x_auth_vesting_types.Periods `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3,castrepeated=github.com/cosmos/cosmos-sdk/x/auth/vesting/types.Periods" json:"lockup_periods"`
    // vesting_periods defines the vesting schedule relative to the start_time
    VestingPeriods github_com_cosmos_cosmos_sdk_x_auth_vesting_types.Periods `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3,castrepeated=github.com/cosmos/cosmos-sdk/x/auth/vesting/types.Periods" json:"vesting_periods"`
}
```

如果出现以下情况，则消息内容无状态验证失败：

* `FunderAddress`或`VestingAddress`无效
* `LockupPeriods`和`VestingPeriods`
  * 包含非正长度或数量的句点
  * 不要描述相同的总量

#### `Clawback`[​](https://docs.evmos.org/protocol/modules/vesting#clawback-2) <a href="#clawback-2" id="clawback-2"></a>

```
type MsgClawback struct {
    // funder_address is the address which funded the account
    FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
    // account_address is the address of the ClawbackVestingAccount to claw back from.
    AccountAddress string `protobuf:"bytes,2,opt,name=account_address,json=accountAddress,proto3" json:"account_address,omitempty"`
    // dest_address specifies where the clawed-back tokens should be transferred
    // to. If empty, the tokens will be transferred back to the original funder of
    // the account.
    DestAddress string `protobuf:"bytes,3,opt,name=dest_address,json=destAddress,proto3" json:"dest_address,omitempty"`
}
```

如果出现以下情况，则消息内容无状态验证失败：

* `FunderAddress`或`AccountAddress`无效
* `DestAddress`不为空且无效

#### `UpdateVestingFunder`[​](https://docs.evmos.org/protocol/modules/vesting#updatevestingfunder) <a href="#updatevestingfunder" id="updatevestingfunder"></a>

```
type MsgUpdateVestingFunder struct {
    // funder_address is the current funder address of the ClawbackVestingAccount
    FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
    // new_funder_address is the new address to replace the existing funder_address
    NewFunderAddress string `protobuf:"bytes,2,opt,name=new_funder_address,json=newFunderAddress,proto3" json:"new_funder_address,omitempty"`
    // vesting_address is the address of the ClawbackVestingAccount being updated
    VestingAddress string `protobuf:"bytes,3,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

如果出现以下情况，则消息内容无状态验证失败：

* `FunderAddress`或无效`NewFunderAddress`​`VestingAddress`

#### `ConvertVestingAccount`[​](https://docs.evmos.org/protocol/modules/vesting#convertvestingaccount) <a href="#convertvestingaccount" id="convertvestingaccount"></a>

```
type MsgConvertVestingAccount struct {
    // vesting_address is the address of the ClawbackVestingAccount being updated
    VestingAddress string `protobuf:"bytes,2,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

如果出现以下情况，则消息内容无状态验证失败：

* `VestingAddress`是无效的

### AnteHandlers <a href="#antehandlers" id="antehandlers"></a>

该`x/vesting`模块提供`AnteDecorator`以递归方式链接在一起形成单个的[`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md)。这些装饰器对以太坊执行基本的有效性检查，以便可以将其从交易 Mempool 中抛出。

请注意，  和 `AnteHandler` 都会被调用 ，因为 CometBFT 提议者目前有能力将失败的交易纳入他们提议的区块中 。`CheckTxDeliverTxCheckTx`

#### Decorators <a href="#decorators" id="decorators"></a>

以下装饰器实现了代币委托和执行 EVM 交易的归属逻辑。

**`EthVestingTransactionDecorator`**[**​**](https://docs.evmos.org/protocol/modules/vesting#ethvestingtransactiondecorator)

根据追回归属账户的归属计划是否已超过归属悬崖和第一个锁定期，验证该账户是否被允许执行以太坊交易。此外，验证该账户是否有足够的解锁代币来执行交易。如果出现以下情况，此 AnteHandler 装饰器将失败：

* 该消息不是`MsgEthereumTx`
* 找不到发件人账户
* 发件人账户不是`ClawbackVestingAccount`
* 区块时间在超越归属悬崖之前（归属币为零）并且
* 区块时间是在超过所有锁定期之前（非零锁定硬币）
* 发送者账户没有足够的解锁代币来执行交易

#### Custom Staking Module <a href="#custom-staking-module" id="custom-staking-module"></a>

Sofa 引入了EVM 扩展的概念，允许智能合约与 Cosmos SDK 模块（如质押和分发）进行交互，从而提供更好的开发人员体验，让用户通过 EVM 与 Cosmos 原生模块进行交互。由于只`ClawbackVestingAccount`允许质押未锁定和已归属的代币，或锁定和已归属的代币，因此我们必须确保所有其他配置都不允许执行状态转换。Sofa`AnteHandler`核心不是在 Cosmos 交易和以太坊交易的 s 中实现这些检查，而是包装 Cosmos SDK`x/staking`模块以在此模块的 s 中引入这些检查`MsgServer`。通过这种方法，我们确保所有质押操作（通过直接 Cosmos 消息或通过扩展）都以正确的方式验证帐户余额。

质押包装器使用与原始质押模块相同的功能，但在以下方法中引入了所需的检查：

* `Delegate`
* `CreateValidator`

### Event <a href="#events" id="events"></a>

该`x/vesting`模块发出以下事件：

#### Create Clawback Vesting Account <a href="#create-clawback-vesting-account-1" id="create-clawback-vesting-account-1"></a>

| Type                              | Attibute Key | Attibute Value         |
| --------------------------------- | ------------ | ---------------------- |
| `create_clawback_vesting_account` | `"funder"`   | `{msg.FunderAddress}`  |
| `create_clawback_vesting_account` | `"sender"`   | `{msg.VestingAddress}` |

#### ClawbackFund Vesting Account <a href="#fund-vesting-account" id="fund-vesting-account"></a>

| Type                   | Attibute Key   | Attibute Value             |
| ---------------------- | -------------- | -------------------------- |
| `fund_vesting_account` | `"funder"`     | `{msg.FunderAddress}`      |
| `fund_vesting_account` | `"coins"`      | `{vestingCoins.String()}`  |
| `fund_vesting_account` | `"start_time"` | `{msg.StartTime.String()}` |
| `fund_vesting_account` | `"account"`    | `{msg.VestingAddress}`     |

#### Clawback <a href="#clawback-3" id="clawback-3"></a>

| Type       | Attibute Key    | Attibute Value         |
| ---------- | --------------- | ---------------------- |
| `clawback` | `"funder"`      | `{msg.FromAddress}`    |
| `clawback` | `"account"`     | `{msg.AccountAddress}` |
| `clawback` | `"destination"` | `{msg.DestAddress}`    |

#### Update Clawback Vesting Account Funder <a href="#update-clawback-vesting-account-funder-1" id="update-clawback-vesting-account-funder-1"></a>

| Type                    | Attibute Key   | Attibute Value           |
| ----------------------- | -------------- | ------------------------ |
| `update_vesting_funder` | `"funder"`     | `{msg.FromAddress}`      |
| `update_vesting_funder` | `"account"`    | `{msg.VestingAddress}`   |
| `update_vesting_funder` | `"new_funder"` | `{msg.NewFunderAddress}` |

### Client <a href="#clients" id="clients"></a>

用户可以`x/vesting` 使用 CLI、gRPC 或 REST 查询 Sofa 模块。

#### CLI <a href="#cli" id="cli"></a>

`sofad` 以下是随模块添加的命令列表 `x/vesting` 。您可以使用该命令获取完整列表 `sofad -h` 。

**Genesis**

genesis 配置命令允许用户配置 genesis `vesting`账户状态。

`add-genesis-account`

允许用户在创世时设置追回归属账户，由代币分配提供资金，并受追回约束。必须提供锁定期文件 ( `--lockup`)、归属期文件 ( `--vesting`) 或两者。

如果提供了两个文件，则它们必须描述相同总金额的计划。如果省略了一个文件，它将默认为立即解锁或归属全部金额的计划。所述数量的硬币将从 --from 地址转移到归属账户。未归属的硬币可能会被出资者使用 clawback 命令“收回”。如果硬币被锁定或未归属，则不得将其转出账户。只有归属的硬币才可以质押。

```
sofad add-genesis-account ADDRESS_OR_KEY_NAME COIN... [flags]
```

**Queries**

该 `query` 命令允许用户查询 `vesting`账户状态。

**`balances`**

允许用户查询给定归属账户的锁定、未归属和已归属代币

```
sofad query vesting balances ADDRESS [flags]
```

**Transactions**

这些 `tx` 命令允许用户创建和收回 `vesting`账户状态。

**`create-clawback-vesting-account`**

如果发送方账户 ( ) 尚未属于此类，则将为其创建一个新的追回归属账户`--from`。只有指定的出资人才能定义锁定和归属计划，并且必须使用 fund-vesting-account 子命令来执行此操作。通过第二个参数启用或禁用通过治理进行的追回。

```
sofad tx vesting create-clawback-vesting-account FUNDER_ADDRESS ENABLE_GOV_CLAWBACK --from=VESTING_ADDRESS [flags]
```

**`fund-vesting-account`**

允许出资人账户使用新计划更新追回归属账户。任何现有计划都会与新添加的计划合并。必须提供锁定期文件 (--lockup)、归属期文件 (--vesting) 或两者。

如果提供了两个文件，则它们必须描述相同总金额的计划。如果省略了一个文件，它将默认为立即解锁或归属全部金额的计划。所述数量的硬币将从 --from 地址转移到归属账户。未归属的硬币可能会被出资者使用 clawback 命令“收回”。如果硬币被锁定或未归属，则不得将其转出账户。只有归属的硬币才可以质押。

```
sofad tx vesting fund-vesting-account VESTING_ADDRESS --from=FUNDER_ADDRESS [flags]
```

**`clawback`**

允许将所有未归属的代币从 ClawbackVestingAccount 中转移出去。必须由原始出资人地址 (--from) 请求，并可以提供目标地址 (--dest)，否则代币将退还给出资人。委托或解绑的质押代币将以委托或解绑状态转移。接收者容易受到削减，如果需要，必须采取行动解绑代币。

```
sofad tx vesting clawback VESTING_ADDRESS --from=FUNDER_ADDRESS [flags]
```

**`update-vesting-funder`**

允许用户更新现有项目的资助者`ClawbackVestingAccount`。必须通过原始资助者地址 ( `--from`) 进行请求。

```
sofad tx vesting update-vesting-funder VESTING_ADDRESS NEW_FUNDER_ADDRESS --from=FUNDER_ADDRESS [flags]
```

**`convert`**

允许用户将其持有的账户转换为链的默认账户（即`EthAccount`）。仅当帐户中没有未持有的代币时，此操作才会成功。

```
sofad tx vesting convert VESTING_ADDRESS [flags]
```

#### gRPC <a href="#grpc" id="grpc"></a>

**Queries**

| Verb   | 方法                                    | 描述             |
| ------ | ------------------------------------- | -------------- |
| `gRPC` | `sofa.vesting.v2.Query/Balances`      | 获得锁定、未归属和归属的代币 |
| `GET`  | `/sofa/vesting/v2/balances/{address}` | 获得锁定、未归属和归属的代币 |

**Transactions**

| Verb   | 方法                                                    | 描述          |
| ------ | ----------------------------------------------------- | ----------- |
| `gRPC` | `sofa.vesting.v2.Msg/CreateClawbackVestingAccount`    | 创建追回归属账户    |
| `gRPC` | sofa`.vesting.v2.Msg/FundVestingAccount`              | 为追回归属账户提供资金 |
| `gRPC` | `/sofa.vesting.v2.Msg/Clawback`                       | 执行追回        |
| `gRPC` | `/sofa.vesting.v2.Msg/UpdateVestingFunder`            | 更新归属账户资金提供者 |
| `gRPC` | `/sofa.vesting.v2.Msg/ConvertVestingAccount`          | 将归属账户转回普通账户 |
| `GET`  | `/sofa/vesting/v2/tx/create_clawback_vesting_account` | 创建追回归属账户    |
| `GET`  | `/sofa/vesting/v2/tx/fund_vesting_account`            | 为追回归属账户提供资金 |
| `GET`  | `/sofa/vesting/v2/tx/clawback`                        | 执行追回        |
| `GET`  | `/sofa/vesting/v2/tx/update_vesting_funder`           | 更新归属账户资金提供者 |
| `GET`  | `/sofasofa/vesting/v2/tx/convert_vesting_account`     | 将归属账户转回普通账户 |
{% endtab %}
{% endtabs %}
