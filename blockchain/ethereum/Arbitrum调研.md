# Arbitrum

> 看东西要带着脑子去看。无论是看代码，还是看文字资料。不要只是眼睛看了，脑子要随时跟着，随时去想去总结。

以太坊扩容方案目前最火热的当属Layer 2，Layer 2又包含默认往以太坊主网上提交的交易有效的Optimistic Rollup和往主网提交交易时会附带其有效性证明的ZK Rollup。

其中，Optimistic Rollup如其名称一样，乐观认为提交至Layer 1主网上的交易有效，验证者可以提出挑战，排序者（往主网提交交易的节点）应答挑战。如果挑战成功，那么提出挑战的验证者将失去其质押，如果挑战失败，那么交易回退，排序者将失去其质押。这一挑战的过程称为欺诈证明（Fraud Proof）。

在欺诈证明的实现细节里，又分为单轮交互的Optimism和多轮交互的Arbitrum。其中，Arbitrum因为其更优的性能表现赢得头筹。

Arbitrum维护了两条链，一条是Rollup方案的Arbitrum One；另一条是AnyTrust方案的Arbitrum Nova，只将数据可用性证书存放在L1上，数据本身不存放在L1上。

单轮交互即当验证者发起挑战时，在Layer 1中一次性验证所有的交易是否正确，比如：Layer 2 提交交易Txn（由提交至Layer2的多个交易汇合而成，这就是Rollup的含义）至Layer 1前的链上状态为State1，交易执行后的状态为State2，那么单轮交互就是只交互一次，Layer 2的排序者将State1、Txn、State2都发给Layer1，由Layer1中的验证节点依次执行Txn中包含的所有交易，从而验证结果是否和State2符合。

多轮交互式欺诈性证明即当验证者发起挑战时，排序者先将交易进行二分协议分割，分为Txn-1和Txn-2，验证者先选一部分进行验证，从而选出出错的一部分，再对出错的一部分再进行二分协议分割，依次递归，直到选中出错的原子交易，挑战结束。这样可以大大减少欺诈证明的成本，但缺点是实现复杂。Arbitrum在实际应用中，采用的是多分协议。

官网：[https://arbitrum.io/](https://arbitrum.io/)

开发者文档：https://docs.arbitrum.io/for-devs/useful-addresses

https://docs.arbitrum.io/proving/challenge-manager

合约代码：[GitHub - OffchainLabs/nitro-contracts: The core Arbitrum Nitro contracts deployed to the base chain to host the rollup](https://github.com/OffchainLabs/nitro-contracts)

合约代码分析：

bridge/SequencerInbox.sol：

从定序器（sequencer）接收批量交易，然后将它们添加到rollup inbox中。

该合约包含收件箱累加器（inbox accumulator），它是rollup要处理的所有数据和交易的排序。作为提交批次的一部分，排序器（sequencer）还应包含延迟收件箱（delayed inbox，在Bridge.sol）中排队的项目。如果排序器（sequencer）未在时限内将delayed inbox中的项目包含在内，任何人都可以强制将这些项目包含到rollup inbox中。

1.  createChallenge()：

challenge/ChallengeManager.sol、rollup/RollupUserLogic.sol。

2.  completeChallenge():

rollup/RollupUserLogic.sol。

3.  bisectExecution()、requireValidBisection()、completeBisection()：

challenge/ChallengeManager.sol、

会触发Bisected()事件。

4.  proveOneStep()：

osp/OneStepProofEntry.sol。

根据不同的情况，选择执行osp/OneStepProver0.sol、osp/OneStepProverHostlo.sol、osp/OneStepProverMath.sol、osp/OneStepProverMemory.sol中的executeOneStep()方法。

在challenge/ChallengeManager.sol的oneStepProveExecution()中调用proveOneStep()。

查看网络上“Arbitrum合约代码分析”相关的资料[https://github.com/dysquard/Arbitrum_Doc_CN/blob/master/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E5%8D%8F%E8%AE%AE/%E6%B4%9E%E6%82%89Arbitrum.md]() 可以了解这些信息：

1.  Aibitrum的多轮交互式欺诈证明的优点就是将大部分工作量放到链下解决，而不是把所有的工作量都放到L1的仲裁合约上。那么同理也能想到，FileDNS的文件验证合约里，欺诈证明的多轮交互式验证这一块，合约逻辑应该是比较简单的，合约代码应该是比较少的，毕竟大部分工作量都在链下，只有极少部分是在合约里操作的。

2.  博弈（也就是挑战）分为两个阶段：分割，然后是单步证明。分割会缩窄双方（挑战方和响应挑战方，也就是辩护方）的争议范围，直至有争议的操作只有一条。随后单步证明会决定谁的主张是对的。

但是看完后，还是没有明白多轮交互验证到底是怎么具体做的。只知道Alice提交了信息，Bob提出了反对，那么Alice就需要对这段信息进行分割，假设这段信息需要执行N条指令，那么Alice第一步需要把她的断言从开始（指令还未执行）到结束（N条指令已执行）在中间点切分（N/2条指令已执行），发布在执行了N/2步指令后的状态。Bob只需要验证该状态是正确的还是错误的，若是正确的，则错误的位置在后半部分，若是错误的，则错误的位置在前半部分。再重复之前的分割动作，经过数轮，Alice和Bob的争议点就缩减为了单步操作（单条指令的执行）。自此，Alice必须给EthBridge生成一个单步证明供EthBridge检测，如果检测通过，则Alice是获胜方，如果检测未通过，则Bob是获胜方。

网络资料——Arbitrum源码分析

https://docs.google.com/document/d/1I6WHgh-cx9fR4lysE7FjCvOlCDn8Uecks6JSXP_14Jw/edit?pli=1#heading=h.rkbif19jq0th

带着问题去看合约代码。

Q：Alice向EthBridge（也就是L1上的合约）提交单步证明供其检测，主要执行了哪些操作？

A：Alice，也就是守卫者，或者Bob，也就是挑战者，都可以将单步证明提交给验证者，也就是L1上的ChallengeManager.sol合约，单步证明包括：

● 断言（包含交易执行前后的AVM状态）

● 虚拟机各个部分所必须的元素。例如：

○ 指令栈的头部元素

○ 数据栈用到的元素

之后，验证者执行单步证明，并将计算后的虚拟机状态，和断言中的虚拟机哈希比对。胜者获得对方质押，败者失去自己的质押。

Q：Alice向L1提交交易批次后，Bob提出挑战是调用哪个接口，并执行哪些操作？

A：在有争议的情况下，Bob会向链上提交PendingDisputeAssert，进而触发ChallengeManager.sol/createChallenge()方法，触发InitiatedChallenge事件，开启bisect二分流程。

Q：Bob提出挑战后，Alice需要在链下和Bob进行多轮交互分割协议，从而确定单步操作，之后Alice向链上提交单步证明吗？这中间有什么时间限制吗？

A：也可以由Bob提交单步证明。提交单步证明时会调用challenge/ChallengeManager.sol/isTimeOut()方法判断当前挑战是否已超时。

## 合约部署

节点启动前，需要先部署好合约。L1上的合约需要手动部署，L2上的合约是预编译的。

L2上预编译的合约接口用solidity语言编写，存放在github/OffchainLabs/nitro-contracts/src/precompiles目录下，但是实现是用go代码实现的，位于github/OffchainLabs/nitro/precompiles目录下。

L1上的合约部署入口为nitro/cmd/deploy/deploy.go。输入所需参数，执行nitro/arbnode包中的DeployOnL1()函数在L1上部署合约。DeployOnL1()执行逻辑为：

deployRollupCreator()：deployBridgeCreator->deployChallengeFactory->deployRollupAdminLogic->deployRollupUserLogic->deployRollupCreator->...
