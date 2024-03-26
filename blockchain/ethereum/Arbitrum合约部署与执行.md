# Arbitrum合约部署与执行

作为Ethereum扩容方案Layer 2中Optimistic Rollup类型的代表项目之一，Arbitrum宣传自己是一个旨在扩展以太坊的技术套件，可以使用Arbitrum链来做用户在以太坊上做的所有事情，包括使用Web3应用程序、部署智能合约等，但是用户的交易会更便宜、更快。

因此，这里我们将使用同一份合约，依次在以太坊测试网Goerli、MEMO dev链、Arbitrum测试网Arbitrum Goerli、Optimism测试网Optimistic Goerli上进行合约部署和合约执行，从而观测交易价格、交易速度的区别。

## 测试目的

1.  OP链（Arbitrum、Optimism）是否支持bls，特别是bls12-377

2.  批量交易是否会减少消耗（gasUsed、gasFee），gasUsed计算方式，gasPrice是多少以及变动方式，gasFee计算方式

## 合约

```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;
import "../library/Secp256k1.sol";
// @dev Deployed Gas: 207266
contract ParsePubKey {
    constructor() {}
    // just for test
    address public addr;
    function parsePubKey(string calldata _methodType, bytes calldata pubKeyData) external {
        bytes32 methodType = keccak256(abi.encodePacked("EcdsaSecp256k1VerificationKey2019"));
        if(keccak256(abi.encodePacked(_methodType))==methodType){
            addr = Secp256k1.getAddrFromCompressedPubKey(pubKeyData);
        }
    }
}
```

上述为稍微修改过的DID/did-solidity的ParsePubKey.sol合约。是一个根据以太坊格式的压缩公钥解析出以太坊格式的地址的合约。合约使用到evm预编译合约expmod。

## 测试资源

● 以太坊测试网Goerli：

可在 https://dashboard.alchemy.com/apps/7l1ryvd8uimmrfsa 查看

在alchemy平台获得rpc端口

https://eth-sepolia.g.alchemy.com/v2/LvulnLMUF5RBRPxddMYPqfZcbThMXMDw

进行测试，使用的是goerli最新版本。

区块浏览器：[https://goerli.etherscan.io/](https://goerli.etherscan.io/)

水龙头：[All That Node | Multi-chain API & Dev-tools, Web3 Infrastructure](https://www.allthatnode.com/faucet/ethereum.dsrv)

● Arbitrum-Goerli：

https://goerli-rollup.arbitrum.io/rpc

区块浏览器：[https://goerli.arbiscan.io/](https://goerli.arbiscan.io/)

水龙头：[Get Arbitrum Goerli Testnet LINK Tokens | Chainlink Faucets](https://faucets.chain.link/arbitrum-goerli)

● Optimistic-Goerli

从alchemy平台获取

https://opt-goerli.g.alchemy.com/v2/6zZn86-K_17oR1Rt0xfyhgCpCe8YKR1V

区块浏览器：[https://goerli-optimism.etherscan.io/](https://goerli-optimism.etherscan.io/)

水龙头：[OP Mainnet Gateway](https://app.optimism.io/faucet)

## 单次发送交易的表现

| 操作\gasUsed | memo-dev | Eth-Goerli | Arbitrum-Goerli | Optimism-Goerli |
| ---------- | -------- | ---------- | --------------- | --------------- |
| 部署合约       | 219326   | 219378     | 219326          |                 |
| 解析公钥       | 61199    | 52195      | 52195           |                 |

### 遇到的问题

1.  在Optimism-Goerli上用相同的go代码逻辑部署合约时，会出现  `parsepubkey err:nonce too low: next nonce 1, tx nonce 0` 的问题，再次执行相同交易会出现`deploy parsepubkey err:replacement transaction underpriced`的错误。

经过猜测并验证，发现，是因为用abigen产生的部署合约parsePubKey的函数中，会先部署secp256k1库，再部署parsePubKey，也就是说调用deployParsePubKey()时，会发送两笔交易，在其他的链平台上运行时，geth节点会自动给每笔交易赋值正确的nonce，但是在Optimism Goerli链上运行时，geth节点没有正确给两笔交易赋值正确的nonce，因此会出现这种错误。

所以，我们接下来先换成简单的合约进行测试。

## 合约

```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract Store {
    mapping(address => uint) public count;

    function add(uint num) public {
        count[msg.sender] += num;
    }

    // Getter function for the cupcake balance of a user
    function getCount(address userAddress) public view returns (uint) {
        return count[userAddress];
    }
}
```

## 单次发送交易的表现(js)

| 操作\gasUsed | memo-dev     | Eth-Goerli | Arbitrum-Goerli | Optimism-Goerli |
| ---------- | ------------ | ---------- | --------------- | --------------- |
| 部署合约       | endpoint 502 | 229058     | 229004          | 229058+16204    |
| 调用add      | endpoint 502 | 44014      | 44014           | 44014+4068      |

## 多次发送交易的表现(go)

| 操作\gasUsed | memo-dev     | Eth-Goerli  | Arbitrum-Goerli | Optimism-Goerli                 |
| ---------- | ------------ | ----------- | --------------- | ------------------------------- |
| 部署合约       | endpoint 502 | 136817      | 136789          | 136817+9848                     |
| 调用add      | endpoint 502 | 43701/26601 | 43701/26601     | 43701+4048/26601+4048/4012/4036 |

## 结论

可以发现，使用js部署合约比使用go部署合约消耗的gas普遍（在每条链上）要多一些，猜测原因应该是合约部署时产生的bin数据有一些不同，经过比对两种方式产生的合约bin数据，发现确实通过hardhat编译出的合约bin数据量要大一些。可能是因为两种方式使用的solc版本不一致。

除此之外，还可以发现，同样合约在L1上和L2上部署运行，消耗的gas大致相同。

## gas & transactionFee

### ethereum

transaction fee = base fee + priority fee(小费)

base fee由协议设定，必须要支付；priority fee 是用户设置用来支付给验证者的小费，以便让验证者可以优先选择将自己的交易打包进区块，当网络繁忙时（交易数量多时），用户可以将priority fee设置的高一些，当网络空闲时，可以将priority fee设置的低一些。

> 例如，假设Jordan要向Taylor支付1个以太币，一笔以太币转账需要21000单位的gas，基础费是10 gwei。Jordan支付了2 gwei作为小费。
> 
> 总费用等于：
> 
> 使用的gas单位数 * （base fee + priority fee）
> 
> 其中base fee由协议设置，priority fee是用户设置的支付给验证者的小费。
> 
> 即21000 * （10 + 2）= 252000 gwei （0.000252个以太币）。
> 
> 当Jordan转账时，将从Jordan账户中扣除1.000252个以太币。Taylor的账户增加1.0000个以太币。验证者收到价值0.000042个以太币的小费。0.00021个以太币的base fee被销毁。

此外，还有一个参数maxFeePerGas，叫做最高费用。用户为他们愿意支付的交易执行费用指定最高限额。最高费用必须超过基础费和小费的总和。交易完成后，会将最高费用与基础费和小费总和之间的差额退还给交易发送人。

区块大小：目标区块大小1500万单位gas，最大不得超过3000万单位gas的区块大小上限。（伦敦升级为以太坊引入了大小可变的区块）

参考资料： https://ethereum.org/zh/developers/docs/gas/

### Arbitrum

需要支付两部分费用：L1上的calldata费用；L2上的网络费，包含计算、存储以及其他（在L2收费部分中，ArbOS对执行其特定预编译合约收取额外费用，该费用根据执行调用时使用的特定资源动态定价）。

对于用户来讲，通过调用Arbitrum节点的`eth_estimateGas`获得的值乘以L2的gasPrice（也就是baseFee，Arbitrum One上起始价格为0.1 gwei，Arbitrum Nova上的为0.01 gwei），就可以得出交易成功所需的以太币总量（包含L1和L2花销）。

也就是：

```
transaction fee(TXFEES) = L2 Gas Price(P) * Gas Limit(G)
```

此Gas Limit包括L2计算的Gas和一个额外的缓冲区，用于覆盖在L1上发布包含此交易的批次时Sequencer支付的L1 Gas。

```
Gas Limit(G) = Gas Used on L2(L2G) + Extra Buffer for L1 cost(B)
```

缓冲区考虑了在L1上发布批量和压缩事务的成本。L1上的估计成本是通过以下两个值相乘得到的：

L1S：通过用Brotli压缩交易来估计，此交易在批次中将占用的数据量（byte，一般是calldata大小，Aribitrum会在此基础上自动加上一个固定值，以弥补交易的静态部分，目前是140 Byte）；

L1P：L2对当前L1每字节数据价格（wei/Byte）的估计，会随着时间的推移动态调整（目前L1 calldata非0字节花费16 gas，0字节花费4 gas，L1P等于L1BaseFee乘以16）；

```
L1 Estimated Cost(L1C) = L1 price per byte of data(L1P) * Size of data to be posted in bytes(L1S)
```

因此，

```
Extra Buffer for L1 cost(B) = L1 Estimated Cost(L1C) / L2 Gas Price(P) 
```

综上，

```
TXFEES = P * (L2G + ((L1P * L1S) / P))
```

### Optimism

交易费用由执行Gas费用和L1数据费用组成。

#### 执行Gas费用

OP主网与EVM等价，OP主网上交易使用的Gas与以太坊上同一交易所使用的Gas完全相同。区别是OP主网上的Gas Price比以太坊上的Gas Price低很多。

> 比如，相同的一笔交易，在以太坊上花费100000 gas，那么在OP主网上也将花费100000 gas。

和以太坊一样，交易费用包含基本费用和优先费。优先费可以设置为0。OP主网排序器将优先处理具有较高优先级费用的交易。

> 例如，如果区块基础费用为1 gwei，并且交易指定优先费用为1 gwei，则每单位gas的总价格为2 gwei。

#### L1数据费用

该费用是由于所有OP主网交易的交易数据都发布到以太坊而产生的，主要由以太坊当前的基本费用决定。L1数据费是自动收取的，该费用直接从发送交易的地址中扣除。确切金额取决于交易的大小（Byte）、当前以太坊的gasPrice和几个小参数。

主要参数有：

● RLP编码的签名交易

● 当前以太坊的gasPrice，也就是baseFee

● fixed_overhead，发布交易的固定间接费用（当前设置为188 gas）

● dynamic_overhead，动态开销成本，随交易大小而变化（当前设置为0.684）

L1数据费用计算会精细到区分交易数据的零字节和非零字节。每个0字节花费4个gas，每个非零字节花费16个gas（相比之下，Arbitrum的计算方式没有区分，而是都乘以16）。

```
tx_data_gas = count_zero_bytes(tx_data) * 4 + count_non_zero_bytes(tx_data) * 16
```

计算出交易数据的gas成本后，将应用固定和动态开销值。

```
tx_total_gas = (tx_data_gas + fixed_overhead) * dynamic_overhead               
```

最后，L1数据费用是通过将总gas成本乘以当前以太坊的基本费用来计算的。

```
l1_data_fee = tx_total_gas * ethereum_base_fee
```
