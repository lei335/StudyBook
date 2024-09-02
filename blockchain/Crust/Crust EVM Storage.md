# Crust EVM storage

## 链接

[Crust Files （一个类似于Dcellar的文件上传界面）](https://crustfiles.io/home/)

## 概述

Crust EVM storgae的架构分为三部分：Contract、Orderer、Connector；它们分别负责EVM兼容交互、去中心化存储桥、和主网资源对接。

### The Contract

Crust EVM storage的合约是由Crust Network组织用solidity语言编写的、基于EVM生态的。合约提供了支付结算机制和可升级的Proxy代理调用机制。

### Orderer

Orderer在合约和Crust mainnet之间搭建了一个bridge，负责处理合约信息和Crust mainnet之间的交互。Orderer监控各个EVM链上“The Contract”的events信息，获取按顺序的订单信息。此外，第三方的Orderer服务也可以通过Crust Tech Commit的允许自由地加入或离开。

### Connector

Connector记录Crust mainnet上EVM跨链存储信息。

## How to works

### The Contract

合约提供3个主要功能：

1.  存储接口：针对于EVM生态的用户，100%兼容IPFS存储协议，提供永久存储；

2.  Orderer注册：对EVM storage的提供者开放，这些提供者可以通过合约注册他们的ETH地址以及提供连接Crust主网和EVM storage 合约接口的能力。它主要负责监听存储接口的订单信息并且将存储接口的信息持久化到Crust主网；

3.  Oracle-based 支付：通过链下oracles，将价格投喂到链上合约中。

### Orderers

Orderer负责监听不同EVM链上的Crust storage contract events信息以及将不同链上的存储信息转发到Crust网络中。任何满足下列条件的第三方，都可以自由访问Crust EVM Storage：

1.  Event Listener：监听events

2.  Parser：解析event信息

3.  Extrinsic Broker（外在经纪人）：解析event信息后，在Crust 主网上放置一个存储订单（文件cid，文件size，文件duration信息）

4.  Price Oracle：负责设置合约里的存储价格

### Mainnet Protocol Adaptation

在Crust主网上持久化存储合约信息。

## Economics经济学

Orderer可以通过提供连接不同EVM链与Crust主网的服务挣收益，主要表现在监听合约event信息并作用到Crust主网上。任何人都可以成为Orderer。Orderers需要监控合约来获取orders，然后他们就有机会在Crust网络上完成orders。他们也需要向合约提供实时的价格，最好价格的提供者有最大的机会被选中去解决order然后获得服务费。

# 合约实现

合约链接：[GitHub - crustio/eth-storage-contract: The storage contract on ETH chain](https://github.com/crustio/eth-storage-contract)

用户接口：

1.  getPrice：通过文件size获得price；

2.  placeOrder：提交订单，订单order包含文件cid和文件size，并且需要通过msg.value转账（值应该等于上面获得的相应的price），用于支付固定节点；

3.  placeOrderWithNode：和placeOrder一样，区别是可以指定固定节点；

> 思考：Crust底层存储是IPFS，但是存到ipfs上首先是把所有数据块存到自己本地节点上，然后会从ipfs网络上随机选取一些节点，分散存放一部分数据块，其他节点也是选择性地存取自己感兴趣的数据块，对于不常被检索的数据块，节点也有可能会删除它们以释放存储空间。Crust所以在合约和ipfs中间加了一层Orderer，可以让Orderer竞价存数据或指定Orderer存数据。但是没有存储证明，无法保证确实存了数据。
