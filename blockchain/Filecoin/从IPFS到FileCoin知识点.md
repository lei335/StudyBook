# 从IPFS到FileCoin知识点

## 内容寻址

传统是基于URL形式的位置寻址，缺点冗余高资源浪费、信任成本高（URL失效、不安全的链接等）。

去中心化网络的内容寻址是基于数据内容进行数据定位，我们都可以保存彼此的数据，优点是存储利用率高、信任成本低、更安全。

### 加密哈希

任何大小、类型的数据，通过哈希后，都会得到一个唯一的、固定长度的字符串。比如bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi。当然，这种形式目前并不人类友好（显然，dog.jpg要友好的多），但它更安全，而且也可以通过类似DNS的方式将其变得人类友好起来。

更安全的原因在于：

● 加密哈希是根据数据本身内容得到的

比如，完全相同的两张图片它们的哈希值也相同，看到相同的两个哈希值，就能确定这两张图片的每个像素都相同。

● 加密哈希是独一无二的

如果数据即使只被更改了一个字母，那么得到的哈希值也会完全不同，这将很容易区分出来文件包含不同的数据。

### 去中心化网络的信任

在中心化网络上，我们已经学会了信任具体的一些机构而不信任其他的，但即使我们做的再好，也仍然会有一些其他恶意机构利用相似的位置定位来欺骗我们，比如恶意链接。

在中心化网络上，我们可以根据内容寻址来相信分享的数据，我们也不会知道太多是哪个节点在保存数据，但是哈希会帮助我们识别验证数据的内容。

### 向节点询问内容

在中心化网络上，我们一般是通过某个域名来访问数据的，比如从域名puppies.com处访问图片dog.jpg，一旦这个域名坏掉了，我们就丢失了对图片dog.jpg的访问。

去中心化网络有不同的工作方式，当我们向去中心化网络通过内容寻址（哈希值）访问dog.jpg时，整个网络都会回应我们，如果Alice在线并有该图片，她就会回应我们，如果她不在线，那Bob或者Grace或者其他人也会回应我们。

### 加密哈希和内容标识符（CIDs）

CIDs最初被开发用于IPFS，它是去中心化网络上内容寻址的一个特别格式。

CID是一个单一的识别符，包含一个codec（编码解码器）和一个加密哈希，编码解码器告诉我们用什么格式编码和解码数据，比如数据格式JSON文件、视频文件等。

CID格式：

Codec | Multihash

其中，Multihash格式：

|Hash Type | Hash Value|

### 将数据链接到一起

不管中心化网络还是去中心化网络，每天都会发生链接，比如把一个logo链接到一个主页、一封email链接一份pdf文档、一份文本文件链接一张图片等。IPFS的Merkle DAG项目，采用CIDs技术构建了一个可以交织的网络。

## CID剖析

### CIDv0

最开始的CID以`Qm`开头，这种CID格式被归类为CIDv0。

CIDv0使用multihash来支持多哈希函数。

multihash格式为TLV（type-length-value），数据的哈希值前面带有哈希函数的编码（见[multicodec table](https://github.com/multiformats/multicodec/blob/master/table.csv)），以及哈希值的长度。

这将得到二进制字节，人们读起来并不友好，因此采用`base58btc`函数对其进行编码，就构造出了字符串格式的CID，该CID格式如下：

`QmY7Yh4UquoXHLPFo2XbhXkhBvFoPwmQUSa92pxnxjQuPU`

所有以`Qm`开头的哈希值都是用`base58btc`编码的CIDv0。

### CIDv1

#### Multicodec prefix

当我们读取数据本身的时候，我们怎么知道数据本身的编码方法呢？它有可能是CBOR、Protobuf、纯JSON文件等等。为了解决这个问题，CIDv1引入了又一个前缀，用来表示数据使用的编码方法。

CIDv1的前缀被称作multicodec prefix。在multihash（type-length-value）的前面又加了一个encode method。

CIDv1包含以下模块：

`<multicodec><multihash-algorithm><multihash-length><multihash-hashValue>`

[multicodec 表格](https://github.com/multiformats/multicodec/blob/master/table.csv)

#### Version prefix

但是，为了区分CIDv0和CIDv1，现在的CID是这种格式：

`<cid-version><multicodec><multihash-algorithm><multihash-length><multihash-hashValue>`

其中，cid-version代表CID放入版本0或者1。

#### Multibase prefix

目前CIDv1的二进制格式包含这些信息：

`<cid-version><multicodec><multihash>`

但是二进制格式的CID并不是human-friendly，所以我们需要字符串格式的CID，比如：

`bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi`

在CIDv0中，总是用base58btc编码将二进制的CID转换为字符串格式，但是现在由于环境限制，我们需要支持更多的编码方法。因此，还需要再添加一个前缀。

该前缀叫做multibase prefix，代表了将CID从二进制格式转换为字符串的进制编码，只用于字符串格式的CID中。

现在的CID中，开头的`b`代表了IPFS中默认使用的base编码方式`base32`。

因此，现在CID格式如下：

`<multibase><cid-version><multicodec><multihash>`

[multibase识别符](https://github.com/multiformats/multibase/blob/master/multibase.csv)

### CID解析器

[https://cid.ipfs.tech/](https://cid.ipfs.tech/)

可以解析CID各字段。

可以认为CIDv0的multibase是`base58btc`，multicodec是dag-pb，multihash-algorithm是sha2-256，multihash-length是32（代表32字节，也就是256 bits）。

----



# Filecoin调研

可以说Filecoin是在IPFS之上增加了激励层，从而激励节点提供长期且有保障的数据存储服务。

Filecoin的存储提供商也叫做存储矿工。

## 存储证明

Filecoin的协议和Filecoin Blockchain使得参与者能够信任数据在规定时间段内被正确存储。

存储矿工需要提供他们存储数据的证明，验证者的职责由Filecoin网络中的所有参与者分担。

Filecoin用来验证存储的加密证明被称作Proof of Replication（PoRep）和Proof of Spacetime（PoSt）。

注意这两个加密证明和加密不同。

虽然区块链通常使用工作量证明（即计算或处理能力），但Filecoin的证明是有用存储的证明（一种权益证明）。因为这些证明不必连续运行，所以它们的计算效率更高，对环境的危害更小。

## 准备和传输数据

### 准备要存储的数据

数据先被分块，然后创建IPLD DAG，该IPLD DAG具有一个payload CID，即一个用来表示DAG根的IPFS CID。

IPLD DAG被序列化成一个[CAR文件](https://zhuanlan.zhihu.com/p/148312314)，并且对该CAR文件填充bit从而制作Filecoin Piece（位填充添加额外的位以使piece符合标准大小），该piece有一个独有的piece CID，也被称为CommP（Piece Commitment，中文名：计件承诺）。

### 协商存储协议和传输数据

当客户端与存储矿工协商存储协议时，客户端雇佣矿工存储一段数据（a piece of data），这些数据可能是整个或者部分文件。矿工将这些来自于一个或多个客户端的碎片（pieces）存储在扇区里（sectors），扇区（sector）是Filecoin使用的基本存储单元。扇区有各种大小，客户存储数据，最多可以存储每笔交易（per deal）的最大扇区大小。

一个piece CID与其他交易参数一起包装从而创建交易提案（Deal Proposal），交易CID包含有关数据本身的信息，以piece CID的形式、矿工和客户端的身份以及其他重要的交易细节。

客户将交易提案（Deal Proposal）发给同意存储他们数据的矿工。一旦矿工确认（confirmed），客户就将他们的数据传输给这些矿工。一旦矿工接收完数据并验证交易提案（Deal Proposal）中的piece CID和数据相匹配，矿工就在Filecoin的区块链上发布交易提案（Deal Proposal），让双方都参与交易（deal）。

## Proof of Replication(PoRep)

在复制证明中，存储矿工证明他们正在存储数据的物理唯一拷贝或副本。复制证明只发生一次，即矿工首次存储数据时。

### 填充扇区并生成CommD

存储矿工收到客户的每一条（piece）数据时，都将数据放入扇区（sector）中。扇区（sectors）是Filecoin的基本存储单元，可以包含来自于多个交易（deals）和客户（clients）的片段（pieces）。

一旦一个扇区（sector）满了，就生成一个CommD（Commitment of Data，又名UnsealedSectorCID），CommD代表这个扇区里包含的所有分片 CIDs（piece CIDs）的根节点。

### 密封扇区并生成CommR

接下来进行的操作被称为密封（sealing）。

在密封期间，扇区数据（由CommD识别）通过一系列图形和散列（hash）过程进行编码（encode），以创建唯一的副本。生成的副本的merkle树的根哈希是CommRLast。

然后CommRLast与CommC（复制证明的另一个merkle根输出）一起进行哈希处理从而生成CommR（Commitment of Replication，又名SealedSectorCID），CommR将被记录到公共区块链上。矿工私下也需保存CommRLast以供将来在时空证明中使用，但不会将CommRLast保存到链上。

编码（encode）过程设计得很慢，计算量很大，因此很难欺骗。<mark>（为什么很慢很大就很难欺骗？）</mark>（编码和加密不同，如果要存储私有数据，则必须在将其添加到Filecoin网络之前对其进行加密）。

CommR提供了我们所需要的证据，证明矿工正在存储客户数据的物理唯一副本。如果多个矿工存储相同的数据，或者单个矿工对相同的数据进行多次存储交易，则每个交易将有不同的CommR.

sealing过程还使用zk-SNARK压缩复制证明，从而保持链更小，以便Filecoin网络的所有成员都可以存储它们以进行验证。

## Proof of Spacetime(PoSt)

时空证明（PoSt）需要重复运行，以证明他们正继续随着时间的推移将存储空间专用于相同的数据。

PoSt建立在复制证明期间创建的几个元素之上：数据副本、私有保存的CommRLast和公开的CommR。

首先，PoSt随机选择编码副本的一些叶子节点，并对它们运行Merkle包含证明（inclusion proofs），以表明矿工具有应该存在的特定字节。然后，矿工使用私有存储的CommRLast来证明（不透露其值）他们知道副本的根，该根既与包含证明（inclusion proofs）一致，又可用于派生公开的CommR。

PoSt的最后一步是把这些证明压缩成一个单独的zk-SNARK。

当矿工同意为客户存储数据时，他们被要求提供抵押品。如果他们在合同期间的任何时候未能通过时空证明，他们将受到惩罚。

## zk-SNARK

Filecoin的复制证明（PoRep）和时空证明（PoSt）过程都使用zk-SNARK进行压缩。

zk-SNARK（Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge）代表“零知识简洁非交互式知识论证”。可以将它们视为计算的哈希值，它们让我们证明（prove）一个证明（proof）已经正确完成，而无需透露证明本身的细节或其所基于的基础数据。

创建Filecoin的zk-SNARK的过程在计算上式昂贵的（慢），但是生成的最终产品很小并且验证过程非常快。与原始证明相比，zk-SNARK很小，这使得它们可以有效地存储在区块链中。比如，在Filecoin链上占用数百KB的证明可以使用zk-SNARK压缩到仅192字节。

运行Filecoin节点的每个人都会维护Filecoin链的最新版本以进行验证。在zk-SNARK的帮助下保持每个证明较小，可以最大限度地减少Filecoin网络中每个节点的存储需求，以及验证交易所需的时间长度。

## 验证交易

压缩后，验证存储所需的关键数据将存储在Filecoin链上，每个运行节点的用户都维护一份副本。这使得时空证明能够随着时间的推移定期运行。

作为一个存储客户端，可以运行命令 `lotus client list-deals` 来列出该节点建议的所有存储交易。例如，以下是仅提出一笔交易的节点的结果：

```shell
$ lotus client list-deals
DealCid:    bafyreiefvrrv5j7omqzfersogg4nqzctyzj66rcmkwkbxxx5prvd5sklci
DealId:     2
Provider:   t01000
State:      StorageDealActive
On Chain?:  Y (epoch 59)
Slashed?:   N
PieceCID:   bafk4chzazx6u4luj34azuit37rlylgrcbgkaakqsjt5avsbolxale2igii3q
Size:       1016
Price:      1000000
Duration:   2744
```

● DealCid：交易提案的内容标识符（CID）；

● DealID：交易的唯一ID；

● Provider：存储矿工；

● State：交易的状态。通常在数据被存储和密封后就会是`StorageDealActive`（但是注意，目前即使在交易期限到期或者矿工未通过时空证明之后，该状态仍将保持不变，因此在后一种情况下，参考`slashed`字段非常重要）；

● on chain：指示交易是否已存储在链上，后面epoch表示存储数据的链上纪元，类似于区块号；

● slashed：指示存储提供者是否未通过时空证明（如果矿工停止存储您的数据，该值将更改为Y，矿工将受到惩罚）；

● pieceCID：存储数据的CID，也被称为CommP（Commitment of Piece）；

● size：存储数据的字节数；

● price：存储交易的每个epoch的价格，以Filecoin Token（FIL）的形式；

● duration：商定交易的每epoch的总持续时间（区块链的一次迭代，目前相当于是30秒）；
