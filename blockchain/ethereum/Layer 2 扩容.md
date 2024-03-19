# Layer 2 扩容

以太坊扩容方案大致包含：

* Layer2
  
  * Optimistic Rollup
  
  * ZK Rollup
  
  * State channel
  
  * Plasma

* Sharding切片

* 侧链（桥接技术）

* Layer1改进

其中，链上方案包括分片、Layer 1改进，链下方案包含Layer 2和桥接技术。

多种方案共同存在可实现结果大于部分之和的效果。

## Rollup

对于Layer 2的方案，一般而言，交易会提交给二层网络节点，而非直接提交给一层网络（以太坊主网）。 对于部分解决方案（Rollup），二层网络实例会将它们分组，然后锚定到一层网络，之后它们受一层网络保护且不能更改。

&nbsp;

Layer 2 的Optimistic Rollup和ZK Rollup都属于Rollup方案，即在以太坊主网链下计算交易，并将交易批次汇合成一笔交易，之后将其锚定到一层网络。

&nbsp;

Optimistic Rollup和ZK Rolllup的区别：

* Optimistic Rollup假设交易是有效的，并且在遇到挑战的情况下只通过欺诈证明进行验证；

* ZK Rollup会在二层网络计算其加密有效性证明，并提交到一层网络；

## Optimistic Rollup

目前采用Optimistic Rolllup方案的主要有两个项目：Optimism和Arbitrum。

### Optimism

Optimism采用单轮交互式欺诈证明。

### Arbitrum

Arbitrum采用多轮交互式欺诈证明。Arbitrum目前维护两条链，一条是Rollup方案的Arbitrum One；另一条是AnyTrust方案的Arbitrum Nova，这种方案只将数据可用性证书发布在L1上，数据本身并不存放在L1上。
