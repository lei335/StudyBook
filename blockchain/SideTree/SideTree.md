### SideTree

是一个运行在区块链平台上的，用于去中心化身份管理的协议。

此协议基于下列模块：

* **Sidetree Core**
  
  监听来自底层区块链的交易输入，使用CAS模块提取其中的DID（Decentralized Identifiers）操作，然后组合/验证每个DID的状态。

* **Content Address Storage**
  
  是一个基于哈希计算的存储接口，使用此接口来交换彼此识别的DID操作批次，从而进行本地持久化和网络传播，DIF（去中心化身份组织）成员为此功能选择了IPFS。

* **Blockchain/Ledger Adapter**
  
  适配器中包含了任何需要读写特定区块链的代码，以便解除Sidetree主体对特定区块链的依赖。
