# Optimism上交易费用的计算

---

date: 2024-7-22

ethereum: Trident(v1.14.7)

optimism: op-node/op-stack(v1.7.7); op-geth(v1.101315.2)

---

交易费用由执行Gas费用和L1数据费用组成。

预估Optimism上交易总费用可

## 执行Gas费用

OP主网与EVM等价，OP主网上交易使用的Gas与以太坊上同一交易所使用的Gas完全相同。区别是OP主网上的Gas Price比以太坊上的Gas Price低很多。

> 比如，相同的一笔交易，在以太坊上花费100000 gas，那么在OP主网上也将花费100000 gas。

执行Gas费用由交易消耗的gas（gas used）乘以每gas的价格（gas price，也称为交易费用）计算得出。

和以太坊一样，交易费用包含基本费用`base fee`和优先费`priority fee`。

### 估算Gas消耗

估算交易gas：调用`eth_estimateGas`JSON-RPC方法。和以太坊一样。

示例：

```shell
cast rpc eth_estimateGas '{"from": "0x1c111472F298E4119150850c198C657DA1F8a368", "to": "0xaa9C082109fb5Dd28D9E6Dd924Ba
0875972bAE60", "data": "'$(cast calldata "setNumber(uint256)" 10)'"}' --rpc-url $RPC_URL
```

或者：

```shell
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_estimateGas","params":[{"from":"0x1c111472F298E4119150850c198C657DA1F8a368","to":"0xaa9C082109fb5Dd28D9E6Dd924Ba0875972bAE60","data":"0x3fb5c1cb000000000000000000000000000000000000000000000000000000000000000a"}],"id":1}' -H "Content-Type: application/json" $RPC_URL
```

会得到16进制的数值，通过`cast`工具可转换为10进制。

```shell
cast to-dec 0xa1b2
```

#### 固定Gas消耗的操作

1.只发送ETH，不涉及任何合约调用时，消耗21000 gas

2.常见操作的固定gas消耗

* Transaction initialization: 21000 gas

* Contract creation: 32000 gas

3.其他常见操作

* Add/Sub: 3 gas

* Mul/Div: 5 gas

* Comparison: 3 gas

* SHA3(keccak256): 30 gas + 6 gas per word(256 bits)

* BALANCE: 100 gas

* MLOAD/MSTORE: 3 gas

* SLOAD/SSTORE: 100 gas / 20000 gas / 2900 gas

* LOG: 375 gas + 375 gas per topic + 8 gas per byte of data

* CALL, DELEGATECALL, STATICCALL, CALLCODE: 100 gas

* SELFDESTRUCT: 5000 gas

EVM操作的详细gas消耗见：[EVM Codes - An Ethereum Virtual Machine Opcodes Interactive Reference](https://www.evm.codes/#10?fork=cancun)

### Base Fee

OP的base费用和L1的base费用计费方式相同，都是根据EIP-1559机制计算的，但是OP使用的EIP-1559参数和以太坊用的参数略有不同。

| Parameter                             | OP Mainnet value | Ethereum value(for reference) |
| ------------------------------------- | ---------------- | ----------------------------- |
| Block gas limit                       | 30,000,000 gas   | 30,000,000 gas                |
| Block gas target                      | 5,000,000 gas    | 15,000,000 gas                |
| EIP-1559 elasticity multiplier        | 6                | 2                             |
| EIP-1559 denominator                  | 250              | 8                             |
| Maximum base fee increase (per block) | 2%               | 12.5%                         |
| Maximum base fee decrease (per block) | 0.4%             | 12.5%                         |
| Block time in seconds                 | 2                | 12                            |

调用rpc接口查询当前的gasPrice（baseFee + priorityFee）：

```shell
 cast rpc eth_gasPrice --rpc-url https://optimism-sepolia.drpc.org
```

没有rpc接口可以直接查询当前的base fee，可以通过gasPrice -   priorityFee来粗略计算。

或者通过调用go-ethereum/consensus/misc/eip1559/eip1559.go中的`CalcBaseFee()`函数可得到精准的baseFee。

也可以通过下面的Gas Price Tracker观察到上一个区块的baseFee，再根据公式计算当前区块的baseFee。

#### Gas Price Tracker

[Grafana](https://optimistic.grafana.net/public-dashboards/c84a5a9924fe4e14b270a42a8651ceb8?orgId=1&refresh=5m) 该内容展示最近eth base fee、OP base fee、OP priority fee。

实时eth base fee：[https://etherscan.io/blocks](https://etherscan.io/blocks)

实时eth blob gas price：[https://etherscan.io/txsBlobs](https://etherscan.io/txsBlobs)

也可在[https://optimistic.etherscan.io/](https://optimistic.etherscan.io/)上的交易详情中看到最近的OP主网和OP测试网的base fee。

观测长期eth gas price：[https://etherscan.io/chart/gasprice](https://etherscan.io/chart/gasprice)

#### Base Fee Calculation

OP和以太坊的base fee计算公式相同，只是部分参数不同。都是根据上一个区块的gasUsed和base fee来计算当前区块的base fee。

```
# parent.gasUsed == gasTarget
baseFee = parent.baseFee
# parent.gasUsed > gasTarget
baseFee = parent.baseFee + parent.baseFee * (parent.gasUsed - gasTarget) / gasTarget / denominator
# parent.gasUsed < gasTarget
baseFee = parent.baseFee - parent.baseFee * (gasTarget - parent.gasUsed) / gasTarget / denominator
```

eth base fee常为个位数量级，大约在3 gwei左右。最近一两个月eth average gas price（base fee + priority fee）大约在10 gwei左右徘徊。

OP base fee大约在0.060 gwei左右徘徊，priority fee也是在0.0几到0.00几 gwei的范围浮动。

可见OP的gas price比eth的gas price低**几百倍**的量级。

eth blob gas price当前为0.000001426 Gwei（1426 wei），当天最低为0.000000001 Gwei（1 wei），可见blob的gas price浮动较大，上千倍的浮动。但是还是远远比eth gas price低，大约为10^6量级。（但是sepolia测试网上，此时eth blob gas price已经超过了eth gas price）。

### Priority Fee

优先费可以设置为0。OP主网排序器将优先处理具有较高优先级费用的交易。rpc方法`eth_maxPriorityFeePerGas`可用于估算当前建议的优先级费用，从而快速将交易打包。

> 例如，如果区块基础费用为1 gwei，并且交易指定优先费用为1 gwei，则每单位gas的总价格为2 gwei。

```shell
 cast rpc eth_maxPriorityFeePerGas --rpc-url https://optimism-sepolia.drpc.org
```

### 举例

比如在OP主网上发起一笔ETH转账：

交易hash：0xd090f000e5ff9e29f19c2545d4b348de2e65339a3a15def4eadffa6ec2ccc02b

L2 gas used：21000

L2 base fee：0.060636858 Gwei

L2 priority fee：0.01 Gwei

L2 gas price：L2 base fee + L2 priority fee = 0.070636858 Gwei

**execution gas fee**：21000 * 0.070636858 Gwei = 1483.374018 Gwei

## L1数据费用

该费用是由于所有OP主网交易的交易数据都发布到以太坊而产生的，主要由以太坊当前的base fee/blob data gas price决定。该费用是支付给L2的sequencer的费用，用于将您的交易发布到以太坊。

Ecotone升级之后，OP stack链可以使用blobs发布交易，当前以太坊的blob gas price将在很大程度上决定L1数据费用。

L1数据费是自动收取的，该费用直接从发送交易的地址中扣除。确切金额取决于压缩后交易的大小（Byte）、当前以太坊的gasPrice以及/或者blob gas price和几个小参数。

之前L1数据费用受以太坊`base fee`的影响最大，但随着Ecotone升级，以太坊的`blob base fee`成为配置为使用`blob`的OP stack链的最大影响因素。每个以太坊区块的`base fee`和`blob base fee`都会在OP主网上更新。

L2的`GasPriceOracle`合约会自动跟踪当前以太坊的`base fee`和`blob base fee`，这两个值每次更新之间的波动最大为12.5%。L1数据费用由您的交易被包含在OP主网块中时看到的以太坊gas price决定，因此可能与您估计的L1数据费用不同。

调用`GasPriceOracle`合约的`getL1Fee`方法可以估算交易的L1数据费用。

### 计算公式

每个版本略有不同。版本升级时间表如下。

| Upgrade | Governance Approval | OP Mainnet Activations                                     | OP Sepolia Activations                      |
| ------- | ------------------- | ---------------------------------------------------------- | ------------------------------------------- |
| Fjord   | approved            | Optimistically Wed Jul 10 16:00:01 UTC 2024 (`1720627201`) | Wed May 29 16:00:00 UTC 2024 (`1716998400`) |
| Ecotone | approved            | Thu Mar 14 00:00:01 UTC 2024 (`1710374401`)                | Wed Feb 21 17:00:00 UTC 2024 (`1708534800`) |

#### Bedrock（Ecotone之前）

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

#### Ecotone

引入了blob。影响参数：

● RLP编码的签名交易

● 当前以太坊的`base fee`以及`blob base fee`（从以太坊中继）

● 两个新的标量参数，用来独立缩放`base fee`和`blob base fee`

每个0字节花费4个gas，每个非零字节花费16个gas。当这个值除以16时，可以认为是对压缩后交易数据大小的粗略估计。

```
tx_compressed_size = [(count_zero_bytes(tx_data)*4 + count_non_zero_bytes(tx_data)*16)] / 16
```

接下来，将两个标量应用于基本费用和blob基本费用参数以计算加权天然气价格乘数。

```
weighted_gas_price = 16*base_fee_scalar*base_fee + blob_base_fee_scalar*blob_base_fee
```

L1数据费用为：

```
l1_data_fee = tx_compressed_size * weighted_gas_price
```

OP主网刚升级为Ecotone时，`base_fee_scalar`设置为`dynamic_overhead`，`blob_base_fee_scalar`设置为0，后期有调整。实际链运营商应该根据[Ecotone fee parameter calculator](https://docs.google.com/spreadsheets/d/12VIiXHaVECG2RUunDSVJpn67IQp9NHFJqUsma2PndpE/edit#gid=186414307)得出scalar值并进行设置。

#### Fjord

Fjord L1数据费计算首先使用基于FastLZ压缩交易规模的线性模型来估计交易规模。

```
estimatedSizeScaled = max(minTransactionSize * 1e6, intercept + fastlzCoef*fastlzSize)
```

模型参数 `intercept` 和 `fastlzCoef` 是通过对先前L2交易的数据集进行线性回归分析来确定的，当使用Brotli压缩时，在历史OP主网数据上，最小化针对批量大小变化的均方根误差。这些参数在Fjord中是固定的。 `minTransactionSize` 、 `intercept` 和 `fastlzCoef` 值按1e6缩放。

接下来，使用两个链参数基本费用标量和团块基本费用标量来计算加权天然气价格乘数。

```
l1FeeScaled = baseFeeScalar*l1BaseFee*16 + blobFeeScalar*l1BlobBaseFee
```

两个标量都以1e6缩放。最终的L1数据费为

```
l1Cost = estimatedSizeScaled * l1FeeScaled / 1e12
```

调用OP链在L1上部署的合约`SystemConfig`中的`setGasConfig()`方法，修改两个标量值，从而改变L1数据费用。L2上的`GasPriceOracle`合约会监听事件从而修改合约中保存的两个标量值（大约1分钟的延迟），计算L1数据费用时用到的标量值就是从`GasPriceOracle`合约中获取的。

一些参数：

| Input arg            | Type      | Description                                                       | Value                    |
| -------------------- | --------- | ----------------------------------------------------------------- | ------------------------ |
| `l1BaseFee`          | `uint256` | L1 base fee of the latest L1 origin registered in the L2 chain    | varies, L1 fee           |
| `l1BlobBaseFee`      | `uint256` | Blob gas price of the latest L1 origin registered in the L2 chain | varies, L1 fee           |
| `fastlzSize`         | `uint256` | Size of the FastLZ-compressed RLP-encoded signed tx               | varies, per transaction  |
| `baseFeeScalar`      | `uint32`  | L1 base fee scalar, scaled by `1e6`                               | varies, L2 configuration |
| `blobFeeScalar`      | `uint32`  | L1 blob fee scalar, scaled by `1e6`                               | varies, L2 configuration |
| `intercept`          | `int32`   | Intercept constant, scaled by `1e6` (can be negative)             | -42_585_600              |
| `fastlzCoef`         | `uint32`  | FastLZ coefficient, scaled by `1e6`                               | 836_500                  |
| `minTransactionSize` | `uint32`  | A lower bound on transaction size, in bytes                       | 100                      |

其中`minTransactionSize`是一笔`Transfer`交易的预估大小。

### Scalar

`GasPriceOracle`合约是`L1Block.sol`的代理合约，`L1Block.sol` 合约记录当前L1区块的相关信息，包含`base fee`和`blob base fee`。

`GasPriceOracle.sol`地址：0x420000000000000000000000000000000000000F

`L1Block.sol`地址：

0x4200000000000000000000000000000000000015

由于这些合约都在链浏览器上进行过验证，所以通过链浏览器可以查看该合约当前记录的状态变量。

OP链浏览器：[https://optimistic.etherscan.io/](https://optimistic.etherscan.io/)，搜索0x4200000000000000000000000000000000000015，得到关于`L1Block`合约的所有交易，点击一笔交易进入详情，之后继续点击合约地址（To字段），查看合约信息，点击'Read as Proxy'，即可查看`L1Block`合约的当前状态，包含`base fee`、`blob base fee`、`base fee scalar`、`blob base fee scalar`。

当前optimism sepolia网的scalar值为：7600和862000；optimism主网为：5227和1014213.

或者通过调用rpc查询`base fee`和`blob base fee`：

```shell
# OP Sepolia
cast call 0x420000000000000000000000000000000000000F 'blobBaseFeeScalar()(uint256)' --gas-price 10000000 --rpc-url https://optimism-sepolia.drpc.org
cast call 0x420000000000000000000000000000000000000F 'baseFeeScalar()(uint256)' --gas-price 10000000 --rpc-url https://optimism-sepolia.drpc.org
# OP mainnet
cast call 0x420000000000000000000000000000000000000F 'blobBaseFeeScalar()(uint256)' --gas-price 100000000 --rpc-url https://optimism.llamarpc.com
cast call 0x420000000000000000000000000000000000000F 'baseFeeScalar()(uint256)' --gas-price 100000000 --rpc-url https://optimism.llamarpc.com
```

#### 更新Scalar

`baseFeeScalar`和`blobFeeScalar`的值可以根据计算器进行估值：[[SHARED] OP Stack: Fjord Fee Parameter Calculator - Google 表格](https://docs.google.com/spreadsheets/d/1V3CWpeUzXv5Iopw8lBSS8tWoSzyR4PDDwV9cu2kKOrs/edit#gid=186414307)

更新`baseFeeScalar`和`blobFeeScalar`的方法：[Using Blobs | Optimism Docs](https://docs.optimism.io/builders/chain-operators/management/blobs)

### Blob Base Fee

Ethereum上的`blob base fee`在2024年3月坎昆升级引入blob交易之后被设置为1 wei，这也是最低价格。blob base fee和base fee有不同的计费机制。

blob监控网站：[https://blobscan.com/](https://blobscan.com/)，可以看到最近的blob交易信息，还会显示由哪个机构（比如Optimism、Arbitrum、base、Taiko等）发起的交易。明显看到Taiko（以太坊二层协议zk-rollup）发起blob交易最为频繁。

Optimism会将L2交易数据压缩后批量上传至L1，以L1的blob交易格式发送，Optimism构造了一个地址`0xFF00000000000000000000000000000000000010`（0x10是OP的chain id），该地址被称为`batch inbox address`，L2往L1发起的blob交易的To字段即为该地址；该交易的sender地址也是由Optimism自行配置的固定地址，被称为`batchSenderAddress`，值为：`0x6887246668a3b87F54DeB3b94Ba47a6f63F32985`。

关于blob交易的详细介绍：[EIP-4844: Shard Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844#blob-transaction)

交易的`blob fee`会被烧毁。

每个blob消耗固定的gas——128*1024=2**17=131072 gas，也就是一个字节消耗一个gas。每个L1 block最多包含6个blob，目标值是3个blob。

`blob base fee`计算方式如下：

```shell
# minBlobGasPrice为1wei，blobGaspriceUpdateFraction为3338477
blob base fee = minBlobGasPrice * e ** (excessBlobGas / blobGaspriceUpdateFraction)
```

`blob base fee`计算规则与EIP-1559相似，根据网络中blob的供需决定，超出目标值则增加，少于目标值则减少，相邻区块的blob base fee差额不超过12.5%。目前ethereum的`blob base fee`常为1 wei，一天可能会有上千倍的浮动。但是sepolia测试网的`blob base fee`却已经达到60000 wei左右，有时甚至超过`base fee`。

调用rpc接口查询当前`blob base fee`：

```shell
 # ethereum mainnet
 cast rpc eth_blobBaseFee --rpc-url wss://ethereum-rpc.publicnode.com
 # ethereum sepolia
 cast rpc eth_blobBaseFee --rpc-url https://ethereum-sepolia-rpc.publicnode.com
```

具体使用的是[指数型EIP-1559更新算法](https://dankradfeist.de/ethereum/2022/03/16/exponential-eip1559.html)，代码实现位于`go-ethereum/consensus/misc/eip4844/eip4844.go/CalcBlobFee()`。

### FastlzSize

交易数据经过FastLZ压缩算法压缩后的数据大小。

### 举例

比如在OP主网上发起一笔ETH转账：

交易hash：0xb0677f99f50d4104cb4aa6af12329957d8fa7c2ee076752b959d693ce7760be0

L1 base fee scalar：5227

L1 base fee：3.357154663 Gwei

L1 blob fee scalar：1014213

L1 blob base fee：1 wei

fastLZSize：170 Byte（transfer交易压缩后大小约为170Byte）

**L1 data fee**：28.076555979 Gwei

其中，

estimatedSizeScaled = max(1e8, -42585600+ 836500*174) = 1e8

l1FeeScaled = 83632* l1BaseFee + 1014213* l1BlobBaseFee = 280765559790229 wei

l1Cost = estimatedSizeScaled * l1FeeScaled / 1e12 = 28.076555979 Gwei

## 执行Gas费用和L1数据费用分析

| Tx hash                                                            | Execution Gas Fee      | L1 Data Fee              | L2 Base Fee/Priority Fee/Gas Used        | L1 Base Fee      | L1 Blob Base Fee |
| ------------------------------------------------------------------ | ---------------------- | ------------------------ | ---------------------------------------- | ---------------- | ---------------- |
| 0x043143d565d6aebb77de260bfcba46bd5b32c63769560e98d385473eeb6c05dc | 0.0001037147490436 ETH | 0.000000013849096012 ETH | 0.000284456 Gwei/2 Gwei/51850            | 1.655956561 GWei | 1 Wei            |
| 0x2813c1c5fb0e7815be294bbd3fe4ff03b63f51f798ef78418c4b175ff501267b | 0.000000020403201 ETH  | 0.000000013849096012 ETH | 0.000285031 Gwei/0.00068655 Gwei/21000   | 1.655956561 Gwei | 1wei             |
| 0x5405b62685cbfff385f9bc5d2c9a115a1b4e2d7e6a981615fc10c85870ab6905 | 0.00000021 ETH         | 0.000000013849096012 ETH | 0.000285031 Gwei/0.009714969  Gwei/21000 | 1.655956561 Gwei | 1wei             |
| 0xe2477498a114b6ea29418f041edf91de4c573daad8ca294eb417e72f4c2695dc | 0.000000025534866 ETH  | 0.000000015686449562 ETH | 0.000291033 Gwei/0.000924913 Gwei/21000  | 1.8756516 Gwei   | 1wei             |
| 0x167ad705c0a07e4b41e07241601e426ac613eb44e478b7a79893fb77f6df0af6 | 0.000042012773502 ETH  | 0.000000022023970797 ETH | 0.000608262 Gwei/2 Gwei/21000            | 2.633438156 Gwei | 8wei             |

可以发现，当前L1 Base Fee约是OP主网L2 Base Fee的几千倍，如果L2 Priority Fee设置的和L2 Base Fee同一个量级，那么最终执行Gas费和L1数据费是同一个量级；如果L2 Priority Fee设置的比L2 Base Fee高一个量级，那么执行Gas费就比L1数据费高一个量级。

在L1 Base Fee大约是Blob Base Fee的10的9次方倍时，L1 base fee上涨1倍，blob base fei上涨8倍，L1 data fee也只是会上涨约1倍，不会受到blob base fee的影响。因为根据L1数据费计算公式：l1FeeScaled = 83632* l1BaseFee + 1014213* l1BlobBaseFee，l1BaseFee的系数比l1BlobBaseFee的系数要低两位数量级。只要L1 base fee高于L1 blob base fee1000倍，L1 bob base fee的数倍变化在L1 base fee的变化前几乎毫无影响。
