# The difference between Meeda and other Eth DA projects

| Project  | proof of continuity | verification    | verification needs to download data | need to run verification node |
| -------- | ------------------- | --------------- | ----------------------------------- | ----------------------------- |
| Meeda    | ✔                   | on-chain        | ❌                                   | ❌                             |
| Celestia | ❌                   | off-chain       | ✔                                   | ✔                             |
| EigenDA  | ❌                   | no verification | no verification                     | no verification               |
| NEAR DA  | ❌                   | no verification | no verification                     | no verification               |

**Meeda** has:

1. continuity proof 

2. on-chain verification

3. Meeda's sampling verification does not require downloading data

4. users do not need to run verification nodes

**Celestia** has no continuity proof and no on-chain verification. They are verified off-chain by verification nodes. 

**EigenDA** does not have an effective verification method. After they store the data on the storage node, the storage node returns the signature. EigenDA just sends the signature to the on-chain contract. This method cannot effectively guarantee that the data is available. 

**NEAR DA** keeps data stored for a short period of time (approximately 60 hours), after which the data is discarded without verification. 

In summary, Meeda has a reliable, credible, and concise verification method, and strives to minimize costs. In addition, Meeda can also support the existing light client verification method in the Ethereum community, but Meeda chooses more efficient and concise on-chain verification.

Meeda具有：

1. 持续性证明

2. 链上验证

3. Meeda的抽样验证不需要下载数据

4. 用户不需要运行验证节点

而Celestia没有持续性证明，也没有链上验证，它们是由验证节点在链下进行验证。

EigenDA没有有效的验证方式，他们将数据存放到存储节点上后，存储节点返回签名，EigenDA只是将签名发到链上合约中而已，这种方式不能有效保证数据可用。

NEAR DA保持数据短期存储（大约60小时），之后数据将被丢弃，并且没有验证。

综上，Meeda具有可靠、可信、简洁的验证方式，并且努力将成本降到最低。此外，Meeda也可以支持以太坊社区现有的轻客户端验证方式，但是Meeda选择更加高效、简洁的链上验证。
