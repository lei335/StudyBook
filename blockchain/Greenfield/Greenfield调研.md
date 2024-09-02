# Greenfield调研

从产品层面上，简而言之，BNB Greenfield的愿景是，BNB Greenfield是BNB Chain生态中，继BNB信标链和BNB智能链之后的第三条区块链，在BNB Chain生态系统中提供去中心化存储基础设施，因此用户和去中心化应用程序（DApp）可以创建、存储和交换他们完全拥有的数据。结果形成一个新的数据驱动经济，BNB链生态系统中的所有参与者现在都可以拥有、使用数据并将数据货币化。

Greenfield 主要由两层够成，一个是Greenfield blockchain，一个是 'storage providers' 组成的网络。

Greenfield 项目旨在解决的一个关键问题是提升数据资产的价值，为用户提供更大的自由来创建、拥有、共享、执行和交易他们的数据资产。

## Greenfield Blockchain

Greenfield blockchain 的共识层基于tendermint。

Greenfield blockchain 不支持智能合约，但BSC和Greenfield之间的原生跨链为生态系统带来了可编程性。可在BSC上部署智能合约将bucket、object、group资产化。

Greenfield Blockchain主要维护用户的账本，并且将storage元数据记录为公共区块链状态数据，此外，它还有专为治理而设计的质押逻辑。

## Greenfield 支付模块

greenfield项目包含一个支付模块，用来处理存储和下载的服务费用。Greenfiled链维护一个公共账本，记录每个账户的余额。

用户只需要往支付模块维护的系统账户转账（也叫做充值）和取款，其余的付费逻辑用户是不感知的。具体的计费和付款逻辑由greenfield的支付模块采用流支付方式自动维护。当费用不足时（缓冲判断机制），用户无法上传、修改文件和桶，质押的一部分（缓冲余额）将被扣除，用户下载速率被降级，阈值时间后若还未充值，用户上传的数据将被SP删掉。

greenfield以message的形式维护账户的流支付记录，并将数据保存到greenfield区块链中。

对于支付模块的具体文档介绍：

https://docs.bnbchain.org/greenfield-docs/docs/guide/greenfield-blockchain/modules/billing-and-payment

相应的代码-proto结构定义：

https://github.com/bnb-chain/greenfield/blob/master/proto/greenfield/payment/stream_record.proto

相应的代码-支付模块实现：

https://github.com/bnb-chain/greenfield/tree/master/x/payment/keeper

## Greenfield 角色

### 1.  Validators

Greenfield blockchain作为一个PoS链，有自己的一组validators，它们通过PoS逻辑选举出来。它们参与greenfield blockchain的治理和质押；接收和处理跨链交易（Greenfield和BSC之间跨链）；维护greenfield的元数据，以便用户和SP(storage providers)操作、存储和访问object的存储。

目前只能由validators运营存储数据的`challenge verifier`，Greenfield计划以后会对外开放。

### 2.  Storage Providers(SP)

SP提供存储服务基础设施，处理用户上传和下载数据的请求，充当用户权限和认证的把关人，确保用户数据的安全性和可访问性。将Greenfield blockchain作为分类账和单一事实来源。

### 3.  用户

以去中心化的方式存储和管理自己的数据，由自己控制和拥有。

## Greenfiled经济学

● 用户向SP支付存储费用，将类似[Superfluid的steam支付方式](https://docs.superfluid.finance/superfluid/protocol-overview/in-depth-overview/super-agreements/constant-flow-agreement-cfa)

● Validators赚取交易费用，以将元数据写入链上

● Validators从存储服务中抽取合理比例的费用

● 用户向SP支付数据下载费用

## Greenfield跨链

greenfield 生态系统有三个不同的层：通信层、资源镜像层、应用层。

### 通信层

通信层由一组**Greenfield Relayers**组成。每个验证者Validator都应该运行一个Relayer，每个Relayer都拥有一个BLS私钥。Relayer监测BSC和Greenfield blockchain 上发生的所有跨链事件，relayer使用BLS签名消息从而确认事件，签名信息被称作 'vote' ，然后relayer将 'vote' 广播给其他relayer. 一旦从relayer中收集到了足够的votes，relayer将轮流组装一个跨链包（cross-chain package）**交易**并将其提交到BSC或者Greenfield网络中。

为激励relayers签署跨链包，将会有中继费（relay fee）作为激励，该费用由触发跨链交易的用户支付。

### 资源镜像层

几乎所有跨链包的目的都是依据BSC上的事件改变Greenfield blockchain上资源实体的状态。能够在BSC上进行镜像的资源包括：Account、BNB、Bucket、Object和Group。

BSC和Greenfield blockchain用的是相同的地址方案（以太坊格式地址），两边相同的地址表示同一个账户。

BNB是建立在BSC上的原生代币，Greenfield无法增发BNB，从而保障BNB的总发行量。

Bucket、Object、Group以一种基于ERC-721和ERC-1155标准改变的新BEP(BEP是BSC的代币标准）格式镜像到BSC上作为NFTs，这些NFT带有与资源（即Bucket、Object、Group）相应的元数据信息，BSC上的NFT的所有权代表了这些资源在Greenfield上的所有权，所有权不可转让。

Greenfield定义了对这些资源进行操作的一系列跨链原语，以供应用层的dApp调用：

**Account**：在BSC上创建支付账户；

**BNB**：在BNB和Greenfield之间双向transfer BNB；

**Bucket**：在BSC上创建bucket；将Greenfield上的bucket镜像到BSC；

**Object**：在BSC上创建object；将Greenfield上的object镜像到BSC；在BSC上将objects授权（或撤销权限）给accounts或groups；在BSC上复制objects；在BSC上启动对object的执行；将buckets关联到BSC上的支付账户；

**Group**：在BSC上创建group；将Greenfield上的group镜像到BSC；更改BSC上group成员；退出BSC上的group；

一旦这些跨链原语被用户或智能合约调用，则将触发相应事件，Greenfield Relayers接收这些事件并将它们转发给Greenfield，一旦这一过程出现错误，将触发回调。所以原语的调用者需要为跨链操作和潜在的回调预先支付费用。

### 应用层

由社区在BSC上部署的智能合约组成，使其能够操作被镜像的资源信息。应用层代表了Greenfield生态系统的真正力量和潜力。

## Greenfield功能特点

### 1.  数据所有权

Greenfield 赋予用户全面的数据所有权管理功能。用户可以将其数据的访问、修改、创建、删除和执行权限**授予个人或组**。在 Greenfield 上，用户可以完全控制自己的数据。

https://greenfield.bnbchain.org/docs/guide/greenfield-blockchain/overview.html#how-greenfield-blockchain-works

### 2.  数据到期后降级服务

一旦支付账户的BNB用完，与这些支付账户关联的对象将遭受降级下载服务，即下载

速度和连接数将受到限制。一旦资金转入支付账户，服务质量便可立即恢复。如果长

时间未恢复服务，则SP自行决定清除数据，类似于SP声称停止对某些对象的服务。在

这种情况下，数据可能会完全从 Greenfield 中消失。

> 如果用户未能及时续订，则存在存储数据被永久删除的风险。

https://greenfield.bnbchain.org/docs/guide/concept/billing-payment.html#payment-account

### 3.  网络治理

验证者在网络治理方面也有发言权。他们对与 Greenfield 生态系统未来发展相关的问题进行投票，并根据需要调整各种网络参数。这确保了网络随着时间的推移保持健康和可持续发展，同时满足用户不断变化的需求。

https://greenfield.bnbchain.org/docs/guide/greenfield-blockchain/overview.html#core-of-greenfield-blockchain

### 4.  搜索和发现

市场应该有一个易于使用的搜索和发现系统来帮助买家找到他们需要的数据，这需要一个强大的索引后端系统。

### 5.  跨链通信

● 用户可以在 BSC 和 Greenfield 之间转移 BNB。

● 用户可以将Group、Bucket、Object作为NFT镜像到BSC。

● 用户可以直接通过智能合约管理BSC上的Group、Bucket、Object。

## 用户与Greenfield的交互

> [GitHub - bnb-chain/greenfield-cmd: support cmd tool for Greenfield](https://github.com/bnb-chain/greenfield-cmd) 运行greenfield-cmd查看更详细的交互。

### Bucket

用户可以通过命令行查看当前SP列表，并且可以查看某个SP的详细信息和价格。

用户创建bucket时，可以输入SP的运营地址，从而指定primarySP。如果未指定，则默认为SP列表的第一个。

用户后期可以修改bucket的可见性（私有和公开）、收费额度、付款地址。

### Object

上传：用户在终端执行上传文件命令后，将往链上发送`createObject`交易，并且把文件内容发送到SP，文件应小于2G，数据被封存后则返回上传信息。

Greenfield还支持在bucket上创建文件夹，将文件上传到文件夹中。

下载：用户从SP那里下载数据。

### Group

group是共享相同权限的账户集合，允许将group作为单个实体处理。

用户可以创建group，并指定group中的初始成员。

用户后期可以添加或删除group中的成员。

### 权限

用户可以给账户或group指定上传文件的权限，包含"create", "delete", "copy", "get" , "execute", "list" or "all"。

用户也可以给账户或group指定更新bucket的权限，包含"update", "delete", "create", "list",  "getObj", "createObj"等。

## Greenfield 功能设计

### 验证身份

使用数字签名，用户使用自己的私钥签署交易。

在节点中，所有数据的存储都使用Protocol Buffers序列化。

### 密钥管理

Greenfield chain 仅支持EIP-712结构化交易，即明确显示签名信息的格式。Greenfield chain 实现了与以太坊兼容的eth_chainId、eth_networkId、eth_blockNumber、eth_getBlcokByNumber、eth_getBalance，从而支持连接像metamask这样的钱包，其他RPC方法未实现。

除了可以使用现有的以太坊钱包，也可以通过[greenfield-cosmos-sdk](https://github.com/bnb-chain/greenfield-cosmos-sdk/tree/master/client/keys)实现的密钥管理接口，来创建一个通用的密钥管理后端。

### Gas和Fees

Greenfield blockchain与Cosmos SDK不同，重新设计了gashub模块，根据交易类型和内容计算gas消耗，而不仅仅是根据存储和计算资源的消耗。没有gas price，在交易预执行过程中推断gasPrice。没有block gaslimit，只限制block大小，不超过1MB。

### 存储

● 主 SP 将存储整个目标文件；

● 辅助 SP 将部分目标文件存储为副本；

● 主 SP 将提供对象的所有下载请求。

每个 SP 都可以设置自己的存储价格，并通过链上交易读取价格。而二级SP存储价格是所有SP存储价格的平均值。

### Group设计

用户A创建一个group；

将group与A上传的对象object进行绑定，并且指定操作权限，比如read等；

用户A将用户B添加到该group中；

则用户B拥有对object的read权限。



在BSC上管理group的合约接口设计如下：

每个组group对应一个NFT Token，group id 即为相应的 NFT ID；组中成员将对应于ERC1155 token，ERC1155 token ID 和所属组的 group id 一样。

```sol
interface IGroupHub {
    // 获取组的合约地址
    function ERC721Token() external view returns (address);
    // 获取组中成员的合约地址
    function ERC1155Token() external view returns (address);

    function createGroup(address creator, string memory name) external payable returns (bool);
    function createGroup(
        address creator,
        string memory name,
        uint256 callbackGasLimit,
        CmnStorage.ExtraData memory extraData
    ) external payable returns (bool);

    function deleteGroup(uint256 id) external payable returns (bool);
    function deleteGroup(uint256 id, uint256 callbackGasLimit, CmnStorage.ExtraData memory extraData) external payable returns (bool);

    function updateGroup(GroupStorage.UpdateGroupSynPackage memory synPkg) external payable returns (bool);
    function updateGroup(
        GroupStorage.UpdateGroupSynPackage memory synPkg,
        uint256 callbackGasLimit,
        CmnStorage.ExtraData memory extraData
    ) external payable returns (bool);
}
```

此外，每个Bucket将对应一个NFT，bucket id 即为 NFT token id.

object也是对应到一个NFT合约中，object id 和 NFT Token ID保持一致。

### 权限控制

GreenField 中的资产包括 group、bucket、object ，资产本身的 owner 具有所有权（增加、更改、删除），另外，还规定了额外的权限管理。

#### 通用权限控制

Greenfield 中的所有资产（group、bucket、object）都对应为一个 ERC721 token，它们的一般权限管理也对应于 ERC721 的权限管理。比如对于group的权限管理，除了 owner 外，可以指定某一个 groupID 授权给一个账户（approve(address _approved, uint _tokenID)），也可以将自己的所有 group 授权给一个账户（setApprovalForAll(address _operator, bool _approved)）。

#### 基于角色的权限控制

当需要对操作粒度进行权限控制，比如允许某一账户创建bucket，但不允许其删除bucket时，则引入了基于角色的权限控制。Greenfield 包含了两种权限控制的角色：

```sol
bytes32 public constant ROLE_CREATE = keccak256("ROLE_CREATE");
bytes32 public constant ROLE_DELETE = keccak256("ROLE_DELETE");
```

## 潜在用例

基于BNB Greenfield，可以创建新一波的DApp.

1.  网站托管

2.  个人云存储

3.  区块链数据存储

4.  发布内容

5.  社交媒体

6.  个人数据市场
