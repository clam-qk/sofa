---
description: 主要针对通过了节点质押规则后，会对这部分节点相应的发放一些权益。
---

# 节点质押权益

### **投票权** <a href="#sofatechnologywhitepaperbak-tou-piao-quan" id="sofatechnologywhitepaperbak-tou-piao-quan"></a>

节点拥有对公链治理的投票权，投票权比率取决于节点质押的权重，简单来说节点质押量越多，投票权重越高。

### **收益** <a href="#sofatechnologywhitepaperbak-shou-yi" id="sofatechnologywhitepaperbak-shou-yi"></a>

#### **Gas费收益**

* gas 费按照比例分法
  * 出块节点 40%
  * 基金会 (Community Pool) 30%
  * 烧毁 30%
* &#x20;实现
  1. 每个块收集 gas fee，发送到 fee\_collector 模块地址
  2. fee\_collector 积攒到一定量的时候会发送到 distribution 模块地址
     1. 分给基金会 (community pool) 的份额在 distribution 里
     2. 分给出块节点的份额也在 distribution 里，出块节点可以随时提取
     3.  销毁的部分每个块都销

         ```go
         k.bankKeeper.SendCoinsFromModuleToModule(ctx, k.feeCollectorName, types.ModuleName, sdk.NormalizeCoins(remaining))
         ...
          
         ...
         k.AllocateTokensToValidator(ctx, validator, remaining)
         ```

#### **块收益**

* 出块收益分配给出块节点，单节点的质押量越大，出块的机会越大，收益越高。
*   实现:

    ```
    err := k.bankKeeper.SendCoinsFromModuleToModule(ctx, minttypes.ModuleName, types.ModuleName, coinbaseCollectedInt)
    ...
    err = k.AllocateTokensToValidator(ctx, validator, reward)
    ```

#### **佣金**

&#x20; 节点可以设定并收取委托质押用户的佣金，用户可以委托节点质押代币，节点可以自己设定涌进比率并且按照此比率收取佣金。

### 出块 <a href="#sofatechnologywhitepaperbak-chu-kuai" id="sofatechnologywhitepaperbak-chu-kuai"></a>

#### **打包交易量**

&#x20; 每个区块最大交易数受区块大小和 gas 限制；gas price 根据最近的区块和交易池动态调节

#### **爆块速度**

&#x20; 最快爆块速度为2秒。

### Gas费分配规则 <a href="#sofatechnologywhitepaperbakgas-fei-fen-pei-gui-ze" id="sofatechnologywhitepaperbakgas-fei-fen-pei-gui-ze"></a>

#### **分配比例**

* 40%的Gas费分配给节点。\

* 30%分配给基金会。
* 30%进行销毁。

### 普通用户质押 <a href="#sofatechnologywhitepaperbak-pu-tong-yong-hu-zhi-ya" id="sofatechnologywhitepaperbak-pu-tong-yong-hu-zhi-ya"></a>

#### **分配比例**

* 委托质押：普通用户可以委托节点进行质押代币，质押可随进随出。\

* 提取期限：用户提取质押代币需要21天后才能到账，意味着用户一旦委托节点质押了代币，如果需要取消质押并且提取该代币，那么需要21天后才能实际到帐。
* 质押收益：年化收益率为16.8%。
