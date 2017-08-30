# 以太坊中的账户、交易、Gas和区块Gas Limit

###### 原文链接: http://hudsonjameson.com/2017-06-27-accounts-transactions-gas-ethereum/
###### 作者: Hudson Jameson
###### 翻译&校对: 许昕

这篇文章是用来帮助人们理解以太坊网络上的一些基本概念和体系，包括账户体系、gas、矿工在区块大小设置机制里的角色等。

## 什么是账户？
### 外部拥有账户 vs 合约账户
以太坊中有两种账户

* 外部拥有账户(EOA)
* 合约账户

这个区别在即将到来的[大都会升级中将会被抽象化](https://github.com/ethereum/EIPs/pull/208)。

#### 外部拥有账户

一个外部拥有账户具有一下特性：

* 有一个以太币余额
* 可以发送交易（以太币转账或者激活合约代码）
* 通过私钥控制
* 没有相关联的代码

#### 合约账户
一个合约账户拥有一下特性：

* 有一个以太币余额
* 有相关联的代码
* 代码执行是通过交易或者其他合约发送的call来激活
* 当被执行时 -- 运行在随机复杂度 （图灵完备性）-- 只能操作其拥有的特定储存，例如可以拥有其永久state -- 可以call其他合约

所有以太坊区块链上的行动都是由各账户发送的交易激活。每次一个合约账户收到一个交易，交易自带的参数都会成为代码的输入值运行。合约代码会被以太坊虚拟机（EVM）在每一个参与网络的节点上运行，以作为它们新区块的验证。

## 什么是交易和消息？

### 交易

"交易"这个术语在以太坊里被用来指代一个用来存储消息的被签名数据包在区块链上从一个外部拥有账户发送至另一个账户的过程。

交易包括：

* 这个消息的接收者
* 一个签名，用来证明发送者有意向通过区块链向接收者发送消息
* 价值域 - 从发送方转移到接受方的wei (ether/10^18) 的数量
* 一个可选的数据域，用来储存发送给合约的消息
* 一个GASLIMIT值，代表了这个交易的执行最多被允许使用的计算步骤
* 一个GASPRICE值，代表了交易发送者愿意支付的gas费用。一个单位的gas表示了执行一个基本指令，例如一个计算步骤

### 消息

合约具有发送"消息"到其他合约的能力。消息是一个永不串行且只在以太坊执行环境中存在的虚拟对象。他们可以被理解为函数调用（function calls）。

一个消息包括：

* 明确的消息发送者
* 消息的接收者
* 一个可选的数据域，这是合约实际上的输入数据
* 一个GASLIMIT值，用来限制这个消息出发的代码执行可用的最大gas数量

总的来说，一个消息就像是一个交易，除了它不是由外部账户生成，而是合约账户生成。当合约正在执行的代码中运行了CALL 或者DELEGATECALL这两个命令时，就会生成一个消息。消息有的时候也被称为"内部交易"。与一个交易类似，一个消息会引导接收的账户运行它的代码。因此，合约账户可以与其他合约账户发生关系，这点和外部账户一样。有许多人会误用交易这个词指代消息，所以可能消息这个词已经由于社区的共识而慢慢退出大家的视野，不再被使用。

## 什么是 gas？

以太坊在区块链上实现了一个运行环境，被称为以太坊虚拟机（EVM）。每个参与到网络的节点都会运行都会运行EVM作为区块验证协议的一部分。他们会验证区块中涵盖的每个交易并在EVM中运行交易所触发的代码。每个网络中的全节点都会进行相同的计算并储存相同的值。合约执行会在所有节点中被多次重复，这个事实得使得合约执行的消耗变得昂贵，所以这也促使大家将能在链下进行的运算都不放到区块链上进行。对于每个被执行的命令都会有一个特定的消耗，用单位gas计数。每个合约可以利用的命令都会有一个相应的gas值。[这里列了一些命令的gas消耗](https://docs.google.com/spreadsheets/d/1m89CVujrQe5LAFJ8-YAUCcNK950dUzMQPMJBxRtGCqs/edit#gid=0)。

### gas和交易消耗的gas
每笔交易都被要求包括一个gas limit（[有的时候被称为](https://media.consensys.net/ethereum-gas-fuel-and-fees-3333e17fe1dc)(startGas）和一个交易愿为单位gas支付的费用。矿工可以有选择的打包这些交易并收取这些费用。在现实中，今天所有的交易最终都是由矿工选择的，但是用户所选择支付的交易费用多少会影响到该交易被打包所需等待的时长。如果该交易由于计算，包括原始消息和一些触发的其他消息，需要使用的gas数量小于或等于所设置的gas limit，那么这个交易会被处理。如果gas总消耗超过gas limit，那么所有的操作都会被复原，但交易是成立的并且交易费任会被矿工收取。区块链会显示这笔交易完成尝试，但因为没有提供足够的gas导致所有的合约命令都被复原。所以交易里没有被使用的超量gas都会以以太币的形式打回给交易发起者。因为gas消耗一般只是一个大致估算，所以许多用户会超额支付gas来保证他们的交易会被接受。这没什么问题，因为多余的gas会被退回给你。

### 估算交易消耗

一个交易的交易费由两个因素组成：

* gasUsed：该交易消耗的总gas数量
* gasPrice：该交易中单位gas的价格（用以太币计算）
交易费 = gasUsed * gasPrice

### gasUsed
每个EVM中的命令都被设置了相应的gas消耗值。gasUsed是所有被执行的命令的gas消耗值总和。

如果希望估算gasUsed，可以使用这个[estimateGas的API](http://ethereum.stackexchange.com/q/266/42)

###gasPrice
一个用户可以构建和签名一笔交易，但每个用户都可以各自设置自己希望使用的gasPrice，甚至可以是0。然而，以太坊客户端的Frontier版本有一个默认的gasPrice，即0.05e12 wei。矿工为了最大化他们的收益，如果大量的交易都是使用默认gasPrice即0.05e12 wei，那么基本上就很难又矿工去接受一个低gasPrice交易，更别说0 gasPrice交易了。

###交易费案例
在被允许后，我将使用这个MyEtherWallet团队的例子并借用他们的分析。请参考[他们与gas相关的介绍](https://myetherwallet.groovehq.com/knowledge_base/topics/what-is-gas)。他们还有一个[小页面方便大家把以太币转换成小单位的gas计数单位](https://www.myetherwallet.com/helpers.html)。

你可以将gasLimit理解为你汽车油箱的上限。同时将gasPrice理解为油价。

对于一辆车来说，油价可能是 $2.5（价格）每升（单位）。在以太坊中，就是20 GWei（价格）每gas（单位）。为了填满你的"油箱"，需要 10升$2.5的油 = $25。同样的，21000个20 GWei的gas = 0.00042 ETH。

因此，总交易费将会是0.00042以太币。

发送代币通常需要消耗大约5万至10万的gas，所以总交易费会上升0.001至0.002个ETH。

## 什么是"区块gas limit"?

区块gas limit是单个区块允许的最多gas总量，以此可以用来决定单个区块中能打包多少笔交易。例如，我们有5笔交易的gas limit分别是10、20、30、40和50.如果区块gas limit是100，那么前4笔交易就能被成功打包进入这个区块。矿工有权决定将哪些交易打包入区块。所以，另一个矿工可以选择打包最后两笔交易进入这个区块（50+40），然后再将第一笔交易打包（10）。如果你尝试将一个会使用超过当前区块gas limit的交易打包，这个交易会被网络拒绝，你的以太坊客户端会反馈错误"交易超过区块gas limit"。[以下例子是来自于以太坊StackExhcange的帖子](https://ethereum.stackexchange.com/questions/7359/are-gas-limit-in-transaction-and-block-gas-limit-different)。

目前区块的gas limit是[4,712,357 gas，数据来自于ethstats.net](https://ethstats.net/)，这表示着大约224笔转账交易（gas limit为21000）可以被塞进一个区块（区块时间大约在15-20秒间波动）。[这个协议允许每个区块的矿工调整区块gas limit，任意加减 1/2024（0.0976%）。](https://www.reddit.com/r/ethereum/comments/6g6tww/there_are_hundreds_or_even_thousands_of_pending/dinzrgq/)

### 谁来决定
区块的gas limit是由在网络上的矿工决定的。与可调整的区块gas limit协议不同的是一个默认的挖矿策略，即大多数客户端默认最小区块gas limit为4,712,388。

### 区块gas limit是怎样改变的
以太坊上的矿工需要用一个挖矿软件，例如ethminer。它会连接到一个geth或者Parity以太坊客户端。Geth和Pairty都有让矿工可以更改配置的选项。这里是geth挖矿命令行选项以及Parity的选项。

## 以太坊网络上的"DoS"攻击是什么？

最近有些评论表示以太坊网络正在慢慢减速，变得拥堵甚至无法使用。这些评论把这个减速的过程称为对以太坊网络的"DoS"攻击。当以太坊网络上持续地出现全满区块并且有大量交易在网络上待处理时就会出现所谓的DoS情况。同时，矿工有权利根据交易费选择打包哪些交易。如果当时队列中（交易池中）有上千笔交易正在等待打包，那么就有可能造成几个小时的非正常交易延迟。DDoS可能是恶意的也有可能是非恶意的。

### 恶意的DoS

上个秋天，以太坊被某人或某个团体攻击了，通过大量制造垃圾交易。这次攻击在如下[博客有介绍](https://blog.ethereum.org/2016/10/18/faq-upcoming-ethereum-hard-fork/)：
```
攻击者通过在他们的智能合约中反复的调用某些命令来让客户端难以处理这些计算，但是这些命令都只消耗少量的gas所以调用起来十分廉价。
```

在这次攻击中，矿工[被要求降低gas limit到150万](https://blog.ethereum.org/2016/09/22/transaction-spam-attack-next-steps/)，在后来的[另一次事件中更改到了200万](https://www.reddit.com/r/ethereum/comments/58aelh/attention_miners_recommending_miners_lower_the/)。也有几次其他的事件要求矿工在网络被攻击时降低区块gas limit。

### 非恶意的DoS
非恶意的DoS其实就是当网络面临海量交易时需要比平常更多的时间来处理一笔交易。最近由于ICO的流行，以太坊网络多次被交易填满。Infura的朋友们写过[一篇与此相关的技术分析文章](https://blog.infura.io/when-there-are-too-many-pending-transactions-8ec1a88bc87e)。

## 为什么区块gas limit在区块被填满时不会自动调整？

#####主要原因：矿工们没有使用gas limit动态调整的功能。

以太坊协议中存在着让矿工可以通过投票来决定gas limit的机制，所以区块容量不需要经过硬分叉就可以调整。最初，这个机制和另一个默认策略是绑定在一起的，即矿工默认投票使区块gas limit至少有470万，并且趋向于最近1024个区块gas使用量的1.5倍。这使得区块容量会根据需求来自动上升，同时也有一个可用来防御垃圾交易的限制。

就像"恶意的DoS"部分说的，在历史上有几次矿工因为攻击的原因不得不使用非默认设置来帮助降低攻击造成的影响。但现在的问题是矿池在攻击之后并没有将设置改回默认设置。大约一个月前，[矿工被要求改变gas limit和gas price设置来再次加入gas limit动态调整功能](https://www.reddit.com/r/ethereum/comments/6ehp60/recommendations_to_miners_to_change_gas_limit_and/)。因为最近的代币销售火爆导致很多区块被填满并且区块链交易堵塞。

[ETH Gas Station](http://ethgasstation.info/index.php)是一个人们可以查阅[最新区块gas limit设置的网站](http://ethgasstation.info/minerVotes.php)。

## 矿工需要做什么才能修复这个问题？

矿工可以在Geth或者Parity客户端中更改设置来重启动态gas limit调整。注意：这些设置是在这个Reddit帖子找到的，其实可以被设置的更高（参考这个帖子）。

### Geth
#### 推荐设置

> --gasprice 4000000000 --targetgaslimit 4712388

#### 解释

> --targetgaslimit Target gas limit sets the artificial target gas floor for the blocks to mine (default: “4712388”)
>
> --gasprice Minimal gas price to accept for mining a transactions (default: “20000000000”).
>
>Note: gasprice is listed in [wei](https://www.myetherwallet.com/helpers.html).

### Parity

#### 推荐设置

> --gas-floor-target 4712388 --gas-cap 9000000 --gasprice 4000000000

#### 解释

> --gas-floor-target Amount of gas per block to target when sealing a new block (default: 4700000).
>
> --gas-cap A cap on how large we will raise the gas limit per block due to transaction volume (default: 6283184).
>
> --gasprice Minimum amount of Wei per GAS to be paid for a transaction to be accepted for mining.
>
>Note: gasprice is listed in wei.
> Note 2: --gasprice is a “Legacy Option”

### 其他挖矿设置选项

可以参考CLI选项页面来看看矿工还能如何调整优化设置。

## Resources and Further Reading
* [Eth Gas Station website with Ethereum gas and miner stats.](http://ethgasstation.info/)
* [Ethereum StackExchange for technical questions of all kinds.](https://ethereum.stackexchange.com/)
* [EthDocs Ethereum Documentation (Much of it is outdated, but still good).](http://ethdocs.org/en/latest/)
* [“Ethereum Gas, Fuel and Fees” by Joseph Chow.](https://media.consensys.net/ethereum-gas-fuel-and-fees-3333e17fe1dc)
* [“What is Gas?” by MyEtherWallet.](https://myetherwallet.groovehq.com/knowledge_base/topics/what-is-gas)
* [MyEtherWallet Ether unit conversion tool.](https://www.myetherwallet.com/helpers.html)
* [“When there are too many pending transactions” by Infura.](https://blog.infura.io/when-there-are-too-many-pending-transactions-8ec1a88bc87e)
