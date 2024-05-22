# kzg trusted setup

FileProof运用kzg技术提供数据承诺以及数据证明，进行kzg计算的前提是需要产生SRS（Structured Reference String），SRS可以被公开，但是用来生成SRS的私密数字secret不能被任何人知道。kzg的实现通常需要用到椭圆曲线，不同的椭圆曲线提供不同程度的安全性和开销，目前FileProof用的是bls12381曲线。

可以将SRS理解为一批椭圆曲线上的点，这些点可以用数学形式表示为$(sg_1、s^2g_1、...、s^ng_1)$，其中s是私密的，$g_1$是椭圆曲线上的生成元，也就是说$s^n$乘以$g_1$得到的值是椭圆曲线上的点，而椭圆曲线上的点具有这个特征：知道$s^ng_1$和$g_1$，无法计算出$s^n$。
那么该如何生成SRS呢？怎么保证SRS中隐藏的secret（也就是数学表达式中的$s$）不被其他人知道呢？答案就是使用`trusted setup`。（该概念在加密资产行业中被广泛使用，毕竟，该行业的最主要目的就是消除单一的可信实体，并用由个体运营商组成的大型、不同的网络取代，从而提高对系统的整体信任）

可信设置的想法是让尽可能多的人参与制作SRS，每个人贡献整个secret的一小部分，然后每个人的贡献都被巧妙地混合在一起从而组合成最终的secret，没有人知道其他人贡献了什么。发现组合secret的唯一方法是每个参与者共同合作从而重现它，所以，一旦有至少一个诚实的参与者不泄露他产生的那部分secret，任何人都不会知道完整的secret。这被称为`1-of-N trust model`.

Zcash在2016年举行了第一次`trusted setup`，但是在当时，那是一个非常复杂的过程，所以只有6个学识渊博的人参与。现在的`trusted setup`变得更加简单、更容易访问，使得上千人能够参与。以太坊的可信设置活动被称为`Summoning Ceremony`，是迄今为止最大规模的可信设置，141416个参与者做出了贡献（来源：https://ceremony.ethereum.org/）。

## 问题1：`trusted setup`具体怎么操作？

这里以`Ethereum Summoning Ceremony`，也就是`Ethereum Trusted Setup Ceremony`为例对`trusted setup`具体怎么操作进行讲解。（参考：https://medium.com/topic-crypto/ethereums-kzg-summoning-ceremony-explained-ecb258830826/How Does It Work?）

`Ethereum Trusted Setup Ceremony`像一个传递包裹的游戏，这个游戏有一个协调者，即以太坊基金会运行的一个服务器，协调者负责监督有哪些参与者、对参与者排序以及对参与者的贡献进行验证。每个参与者会生成一个私密随机数，当协调者对参与者说“轮到你了”后，参与者会收到一长串超过4000个数字的列表，用数学公式来详细地表示为：$g_1、s_1s_2...s_i*g_1、(s_1s_2...s_i)^2*g_1、...、(s_1s_2...s_i)^{4095}*g_1$，这是椭圆曲线点的集合，包含所有先前参与者的秘密的组合。这时，参与者可以通过乘法将他的私密数字$s_j$混合到组合中，简单地讲，就是将每个椭圆曲线点乘以$s_j$，用数学公式详细地表示为：$g_1、s_1s_2...s_is_j*g_1、(s_1s_2...s_is_j)^2*g_1、...、(s_1s_2...s_is_j)^{4095}*g_1$。需要注意的是，这些计算都是在有限域上完成的，可以把它的工作原理理解为时钟，当数字达到某个最大值时，会循环回下限，这意味着两个数字相乘可能会产生一个较小的数字。一旦验证者完成该过程，他就会丢弃$s_j$，并且将更新后的数字列表发送回协调员，协调员执行快速验证后，将列表发送给下一个参与者重复该过程，直至最后一个参与者贡献完毕，得到最终的SRS：$g_1、sg_1、s^2g_1、s^3g_1、...、s^ng_1$。除非所有的参与者都泄露出自己的secret，否则将无人知晓最终完整的secret，也就是公式中的$s$。

以太坊KZG召唤仪式的参与者需要同时对两组椭圆曲线点进行上述操作，另外一组是基于椭圆曲线G2域生成元$g_2$进行计算的，因为KZG方案需要两组相关但略有不同的点才能发挥作用。

以太坊针对不同长度的点集进行了上述操作，从而应对未来不同大小的blob需求，具体来讲，G1点个数依次为4096、8192、16384、32768。以太坊KZG召唤仪式的SRS结果保存在：https://github.com/ethereum/kzg-ceremony/blob/main/transcript.json。 在这里可以看到四组不同长度的SRS点集。

需要注意的是，可信设置的操作方式并不是唯一的。以太坊利用`Powes-of-tau`进行可信设置，除此之外，还有其他方式进行可信设置，比如旧版ZK-SNARK协议（详情：https://vitalik.eth.limo/general/2017/02/01/zk_snarks.html）。 但只需要记住，更多参与者总是比更少参与者要好，每个参与者仅需要一轮交互的仪式比需要多轮交互的仪式要好。理想情况下，仪式应该是通用的，一个仪式的输出能够支持多种协议。

## 问题2：FileProof可以直接使用Ethereum的`trusted setup`吗？

FileProof使用的也是基于bls12381椭圆曲线的kzg技术，和以太坊相同（以太坊坎昆升级后的版本）。那么FileProof可以直接使用以太坊的可信设置吗？

目前go-ethereum中使用的可信设置参数位于：https://github.com/ethereum/go-ethereum/blob/master/crypto/kzg4844/trusted_setup.json ，其中$G_1$点使用的是Lagrange格式，无法直接用于FileProof的vk参数赋值（因为目前FileProof使用的是单项式格式的可信设置，meeda-v1.1.0）。如果想直接使用该`trusted_setup.json`文件中记录的拉格朗日格式的$G_1$值，客户端对数据的操作需要与当前go-ethereum中实现的操作完全相同（坎昆升级后的go-ethereum版本。大概包括对$G_1$进行位逆反；不使用系数形式的多项式，而是使用评估形式的多项式）。

使用拉格朗日格式的可信设置（https://github.com/ethereum/go-ethereum/blob/master/crypto/kzg4844/trusted_setup.json）会比使用单项式格式的可信设置（https://github.com/ethereum/kzg-ceremony/blob/main/transcript.json） 进行验证和生成承诺的效率高。该结论的详细讲解可参见下面的链接4.

https://github.com/ethereum/kzg-ceremony/blob/main/transcript.json 文件中记录着原始格式的$G_1$值，可以直接用于FileProof中的vk赋值。

关于拉格朗日格式的可信设置，这里有相关讨论：
1. https://vitalik.eth.limo/general/2022/03/14/trustedsetup.html ，目录`Setups in Lagrange form`
2. https://ethresear.ch/t/kate-commitments-from-the-lagrange-basis-without-ffts/6950
3. https://inevitableeth.com/home/concepts/polynomial-encoding
4. https://medium.com/coinmonks/trusted-setup-in-zksnarks-powers-of-tau-vs-lagrange-basis-7f12978f1eb9
这里不做过多阐述。
