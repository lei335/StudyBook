# Fabric类联盟链的存证实现

## Fabric的介绍

[初识联盟链1: Fabric 是什么](https://yangzhe.me/2020/04/10/fabric-intro/)

> 共享账本
> 
> Hyperledger Fabric 有一个分类帐子系统，包含两个组件：World State和Transaction Log。每个参与者都有一份他们所属的每个 Hyperledger Fabric 网络的账本副本。
> 
> World state组件描述了给定时间点的账本状态。它是分类帐的数据库。Transaction log组件记录所有导致世界状态当前值的交易；这是世界状态的更新历史。那么，分类帐是世界状态数据库和交易日志历史记录的组合。
> 
> 分类账有一个可替换的世界状态数据存储。默认情况下，这是一个 LevelDB 键值存储数据库。事务日志不需要是可插入的。它只是记录区块链网络使用的分类账数据库的前后值。
> 
> 智能合约
> 
> Hyperledger Fabric 智能合约是用ChainCode编写的，当应用程序需要与账本交互时，由区块链外部的应用程序调用。在大多数情况下，ChainCode只与账本的数据库组件、世界状态（例如查询它）交互，而不是事务日志。
> 
> ChainCode可以用多种编程语言实现。目前支持 Go、Node.js 和 Java 链码。

### Ledger

### World State interaction

```sol
// State defines interaction with the world state
type State interface {
	// GetPrivateDataMultipleKeys gets the values for the multiple private data items in a single call
	GetPrivateDataMultipleKeys(namespace, collection string, keys []string) ([][]byte, error)

	// GetStateMultipleKeys gets the values for multiple keys in a single call
	GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error)

	// GetTransientByTXID gets the values private data associated with the given txID
	GetTransientByTXID(txID string) ([]*rwset.TxPvtReadWriteSet, error)

	// Done releases resources occupied by the State
	Done()
}
```

### Deploy chaincode

最终用户通过调用智能合约与区块链分类账进行交互。

将chaincode部署到区块链的channel中需经过四步：

1.  打包智能合约

2.  安装chaincode包

3.  批准chaincode定义

4.  将chaincode定义提交到channel

详细可见：

https://github.com/hyperledger/fabric/blob/main/docs/source/deploy_chaincode.md


