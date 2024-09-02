# ioTex与MEMO合作点

## 存储方面

1.  链数据备份

2.  W3bStream

W3bstream是一个通用框架，用于将物理世界中生成的数据连接到区块链世界。W3bstream使用ioTex区块链来编排一个分散的网关网络（即运行多个W3bstream节点），该网络从IoT设备和机器（比如ioTex的Pebble Tracker硬件设备，通过设备上的传感器获取真实世界的数据）流式传输加密数据，并为区块链生成证明。从而开发人员即可在W3bstream框架上构建能够访问真实世界数据的dapp.

目前，W3bstream Alpha版本仅包含一个简单的键值数据库形式的存储接口实现。W3bstream 后续将提供更多的存储选项，包括集中式和分散式。

MEMO可通过gateway方式为W3bstream提供分散式的存储服务。

3.  DID Doc

ioTex DID允许人们和设备将DID文档和凭证存储在他们选择的存储中，为了缓解DoS攻击，DID所有者可在不同的存储位置存储DID文档或凭证的多个副本。DID Doc可以上传到任何可以公开访问的内容存储，比如S3或IPFS或其他云存储。

ioTex可以鼓励用户或者默认存储到MEMO的中间件服务上。

## 非存储方面

1.  ioTex提供了[Rugdoc](https://rugdoc.io/education/smart-contract-audits/)：一个社区审查的智能合约审计平台。

MEMO可考虑进行智能合约审计。

2.  https://docs.iotex.io/dapp-development/developer-grants

ioText提供开发者资助计划，该计划对任何阶段的项目开放申请，只要该项目促进 IoTeX 技术、生态系统和社区的发展和采用。

MEMO可考虑提交资助申请。
