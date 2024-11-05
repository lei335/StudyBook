# 部署Meeda版OP-Stack-Layer2

---

date: 2024-11-5

op-contracts: op-contracts/v2.0.0-beta.3

op-node: op-stack(v1.9.4);

op-geth: v1.101411.0

---

参考官方文档：[Creating Your Own L2 Rollup Testnet | Optimism Docs](https://docs.optimism.io/builders/chain-operators/tutorials/create-l2-rollup)。以下是纯步骤分享。

## 获取源码

```shell
cd ~/op-layer2
git clone https://github.com/memoio/optimism.git
cd optimism
# 进入正确的分支（当前最新的合约版本，与当前OP Sepolia版本一致）
# git checkout op-contracts/v2.0.0-beta.2
git checkout op-contracts/v2.0.0-beta.3
```

## 安装依赖

```shell
# 检查依赖版本(以下安装版本是当前要求，后续应根据具体要求安装相应版本)
./packages/contracts-bedrock/scripts/getting-started/versions.sh

# 安装go
wget -c https://golang.google.cn/dl/go1.21.1.linux-amd64.tar.gz
tar zxvf go1.21.1.linux-amd64.tar.gz -C /usr/local
cd ~
mkdir go
vim ~/.profile
# 在.profile文件最后添加上：
export GOPATH=$HOME/go
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
# 检查
source .profile
go version
# 可更换下载源
go env -w GOPROXY=https://goproxy.cn,direct

# 安装node
wget https://nodejs.org/dist/v20.16.0/node-v20.16.0-linux-x64.tar.xz
tar -xvf node-v20.16.0-linux-x64.tar.xz
mkdir /usr/local/nodejs
mv node-v20.16.0-linux-x64/* /usr/local/nodejs/
# 在.profile文件最后添加上：
export NODE_HOME=/usr/local/nodejs
export PATH=$NODE_HOME/bin:$PATH
# 检查
source .profile
node --version

# 安装pnpm
npm install -g pnpm
pnpm --version

# 安装foundry
curl -L https://foundry.paradigm.xyz | bash
# 根据上面命令的输出执行接下来的操作
source .profile
foundryup
# 检查
forge --version

# 安装jq
sudo apt-get install jq
jq --version

# 安装direnv
curl -sfL https://direnv.net/install.sh | bash
# 在.bashrc文件最后一行添加：
eval "$(direnv hook bash)"
# 检查
direnv --version

# 安装make
sudo apt-get install make

# 安装just
curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/
vim ~/.profile
# 在.profile文件最后添加上：
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin:/usr/local/
# 检查
source .profile
just version
```

之后安装其余依赖项并编译合约：

```shell
# cd ~/op-layer2/optimism
# pnpm install
# pnpm build
cd ~/op-layer2/optimism/packages/contracts-bedrock
just build
```

> 以下内容已过时，因为op不再使用pnpm，转而使用just：
> 
> 执行`pnpm install`命令可能会遇到错误：
> 
>  `ERR_SOCKET_TIMEOUT  request to https://registry.npmjs.org/@react-native/debugger-frontend/-/debugger-frontend-0.73.3.tgz failed, reason: Socket timeout`
> 
> 建议解决方案：
> 
> ```shell
> # 方案一
> # 如果网络连接很糟糕，可能需要设置成更大的值（在后面添加1个或多个0）
> npm config set fetch-retry-mintimeout 20000
> npm config set fetch-retry-maxtimeout 120000
> # 想查找这两个配置的默认值
> npm config ls -l
> 
> # 方案二
> # 更改淘宝镜像
> npm config set registry http://registry.npmmirror.com
> 
> # 方案三
> # 删除缓存
> npm cache clean --force
> 
> # 方案四
> # 清除代理，关机重启
> npm config rm proxy
> npm config rm https-proxy
> # 还原registry
> npm config set registry http://registry.npmjs.org/
> ```
> 
> 在上面这个问题上花费了很多时间，只能多次重试，最终`pnpm install`成功执行，所有依赖全部安装完成。（这里我只用了方案一）

## 填写环境变量

```shell
cd ~/op-layer2/optimism
# git checkout op-contracts/v2.0.0-beta.2
git checkout op-contracts/v2.0.0-beta.3
cp .envrc.example .envrc
# 执行后会报错，暂时不用理会

# 打开.envrc文件并填写相应内容。举例：
# 该rpc从infura申请，延用了旧版memo-L2的参数（位于gitlab/middleware/memoda/blob/master/docs/opstack/params.md）
L1_RPC_URL = https://sepolia.infura.io/v3/1b65bb7a429c4970b5f3381ede10f8e7
L1_RPC_KIND = infura

L2_CHAIN_ID = 11155986 # 随机设置一个不一样的数值，每条链都应使用不同的chain_id
```

## 生成地址

测试环境可以通过脚本生成：

```shell
./packages/contracts-bedrock/scripts/getting-started/wallets.sh
```

生产环境需要使用硬件安全模块和硬件钱包的组合。

这里继续使用旧版memo-L2的参数：

```shell
# Admin address
export GS_ADMIN_ADDRESS=
export GS_ADMIN_PRIVATE_KEY=

# Batcher address
export GS_BATCHER_ADDRESS=
export GS_BATCHER_PRIVATE_KEY=

# Proposer address
export GS_PROPOSER_ADDRESS=
export GS_PROPOSER_PRIVATE_KEY=

# Sequencer address
export GS_SEQUENCER_ADDRESS=
export GS_SEQUENCER_PRIVATE_KEY=
```

将这四个账户地址和私钥保存到.envrc文件中。

并给这些账户充值，官网建议的资金为：admin - 0.5 sepolia ETH、proposer - 0.2 sepolia ETH、batcher - 0.1 sepolia ETH

## 加载环境变量

```shell
direnv allow
# 正确执行会有类似如下输出：
direnv: loading ~/optimism/.envrc                                                            
direnv: export +DEPLOYMENT_CONTEXT +ETHERSCAN_API_KEY +GS_ADMIN_ADDRESS +GS_ADMIN_PRIVATE_KEY +GS_BATCHER_ADDRESS +GS_BATCHER_PRIVATE_KEY +GS_PROPOSER_ADDRESS +GS_PROPOSER_PRIVATE_KEY +GS_SEQUENCER_ADDRESS +GS_SEQUENCER_PRIVATE_KEY +IMPL_SALT +L1_RPC_KIND +L1_RPC_URL +PRIVATE_KEY +TENDERLY_PROJECT +TENDERLY_USERNAME
```

## 配置网络

```shell
cd packages/contracts-bedrock
# 生成配置文件
./scripts/getting-started/config.sh
# 在packages/contracts-bedrock/deploy-config目录下会生成一个文件：getting-started.json

vim deploy-config/getting-started.json

# 修改getting-started.json文件：
## 修改以下值
"batchInboxAddress": "0xff00000000000000000000000000000011155986" # 随机设置，一般后缀和chainID一致
# "l2OutputOracleSubmissionInterval": 120 # 初始值为120，表示多久向L1提交output root, 120 * L2BlockTime
"eip1559Denominator": 250
# "faultGameGenesisOutputRoot": "0x0000000000000000000000000000000000000000000000000000000000000000",

## 添加以下内容
# "l2GenesisFjordTimeOffset": "0x0", # 表示激活Fjord版本
# "L2OutputOracleStartingBlockNumber": 0,
"gasPriceOracleBaseFeeScalar": 7663,
"gasPriceOracleBlobBaseFeeScalar": 0,
"useFaultProofs": false, 
"disputeGameFinalityDelaySeconds": 600,
"proofMaturityDelaySeconds": 600,
"respectedGameType": 0,
"useAltDA": true,
"daCommitmentType": "GenericCommitment",
"daChallengeWindow": 300,
"daResolveWindow": 300,
```

生产环境中，l2ChainID应该设置为唯一的值，并且应该将该chainID添加到[ethereum-lists/chains](https://github.com/ethereum-lists/chains)。

`useAltDA`设置为true时，会部署DataAvailabilityChallenge合约，支持用户对使用AltDA的指定commitment进行挑战，这部分逻辑称为Plasma，但是现在Plasma只支持keccak256类型的commitment，与Meeda不兼容，因此目前并不需要应用plasma，Meeda本身就支持挑战和证明，所以不应用Plasma并不会造成安全隐患。

`useFaultProofs`设置为true时，表示激活Fault proof，此时将使用OptimismPortal2.sol而不是OptimismPortal.sol。当不激活故障证明时，由单个op-proposer提交链的存储状态outputRoot，这种方式比较中心化，且高度依赖对op-proposer的信任。目前OP正在开发和测试故障证明，将outputRoot的提交去中心化，引入op-challenger等，无许可的故障证明允许任何人提交outputRoot，但是会有监控器和挑战者对这些outputRoot进行验证、挑战。

> 设置为以上值时，发现链会经常**回滚**，op-node日志报错为：`t=2024-08-21T02:27:27+0000 lvl=warn msg="L2 reorg: existing unsafe block does not match derived attributes from L1" err="random field does not match. expected: 0x6cc733d89ec1fb3314d3b0ddbb1ccdb1d0f1ff8a7125b68a7c1395bd9a30d753. got: 0xe88014e2c77b43a9f0c9ac092eb01a8f366a305b1367af08e3a67b711a6fc7eb" unsafe=0x7e54dbef28b6ffaf9d52c23a870349195a5ce28b57f47f94d982c76ed7f25a76:268974 pending_safe=0x3423856f19d103fd5462d2a47e9768e066f2bc2f0860d8a8db3af45eb0cd44ae:268973`
> 
> 可见错误原因是从L1推导的信息和op-node本地缓存的unsafe区块信息不一致。而往L1上提交的信息只有两部分，一个是proposer定期往L1提交output root，一个是batcher定期把batch数据提交至Meeda，然后把相应的commitment提交到L1，那么就说明这两部分出了问题，导致报错。查看更详细得日志，发现sequencer从L1派生L2区块attributes时，区块中得random参数（也就是该L2区块的origin L1区块中的random参数）和本地生成的L2 unsafe区块中不符合。派生的区块信息是正确的，本地unsafe区块信息是错误的，错误地应用了上一个origin区块的random值。也就是sequencer生成unsafe区块时，错误地采用了旧的origin区块信息，而没有及时更新。
> 
> 目前还没有解决方法，使用最新版本重新部署L2，观察能否解决L2 reorg的问题。

## 部署L1合约

注意：可通过[etherscan](https://sepolia.etherscan.io/blocks?p=1)观察`base fee`大小，选择值较小的时候再部署合约，从而花费更少的eth。

当前部署全部的L1合约需要72525043gas，在gas price预估为3.18 gwei时，预估将花费0.231 ETH.

```shell
cd packages/contracts-bedrock/

# 以免版本冲突
forge cache clean
forge clean

DEPLOYMENT_OUTFILE=deployments/artifact.json DEPLOY_CONFIG_PATH=deploy-config/getting-started.json forge script scripts/deploy/Deploy.s.sol:Deploy --private-key $GS_ADMIN_PRIVATE_KEY --broadcast --rpc-url $L1_RPC_URL --slow
```

之后合约地址会保存在`deployments/artifact.json`文件中。以下是部署的合约示例：

```shell
{
  "AddressManager": "0x4a37bD1DcFcba8c3e0c97CF071bf7997fBcE5613",
  "AnchorStateRegistry": "0x18541C871C2BF2A9f0d246062d6D4e7afE6D0AEd",
  "AnchorStateRegistryProxy": "0x088469Df3Ff7ac004A8BF069E585756109237Be7",
  "DelayedWETH": "0xB76b967599Fbd76c0d551bAF36dc071bFD94229C",
  "DelayedWETHProxy": "0x8b51160dbc460eE6c58a14383b584D931F2bb45B",
  "DisputeGameFactory": "0xBb5026207010148f62bFFa08c74e91f572630a43",
  "DisputeGameFactoryProxy": "0xD90D54685c4536D506a202E40bb211a3a5047696",
  "L1CrossDomainMessenger": "0x4ACe5179eeAc2Fa68FFc5A506eE75eC3BD14A44f",
  "L1CrossDomainMessengerProxy": "0xbe5f9dEFa43f52b482a91a8D3bbc1c8EA987c31E",
  "L1ERC721Bridge": "0xc01837b4953e7Bd72eF12419788a735f1f73B4F2",
  "L1ERC721BridgeProxy": "0x68E194834390A76FC2C449310746248401692f68",
  "L1StandardBridge": "0xA1aC0fCd63848e1E15BB3D14eC3987de9Cf0d6AE",
  "L1StandardBridgeProxy": "0x2bb08a800A11Ef9F1a0EB4A47Ab1caf045315246",
  "L2OutputOracle": "0x1e53298ab6eA5168610d44e291746dA2a4D1e9F5",
  "L2OutputOracleProxy": "0x0A3EeE49381171A960D3451DF42420B5A5a4dd49",
  "Mips": "0x4D0c1177d29c185E656cd1D7D1045EcAC50d3E1a",
  "OptimismMintableERC20Factory": "0x59a9A6528200B7948f7c70BabB05Dd050fcdE38a",
  "OptimismMintableERC20FactoryProxy": "0x867fb6DF3f49336d12476d85f5B8373371b53409",
  "OptimismPortal": "0xEB285cD027A4F8634e75F60bc7FD9FaE02B367Bb",
  "OptimismPortal2": "0xd2Cc50812436cC9bFD330AC47022576E72dE7b6b",
  "OptimismPortalProxy": "0x3a4B63B8EC1596f88f7318b9cE6cE70BA10D70EB",
  "PreimageOracle": "0xb3c54B72e0a95eECb5f804502a4eb34f3F93bC4D",
  "ProtocolVersions": "0x7951Dcb817d20997ab3e3f086Eaa97994b02D95C",
  "ProtocolVersionsProxy": "0xfb4D5062DBBfB419499e78Dc77dd1d47E776B604",
  "ProxyAdmin": "0x231DD86fe5AEF6EF67472677DfD0485c75dF779C",
  "SafeProxyFactory": "0xa6B71E26C5e0845f74c812102Ca7114b6a896AB2",
  "SafeSingleton": "0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552",
  "SuperchainConfig": "0x46D9445Bb910b6ec6EB731187b45D4449456f0e2",
  "SuperchainConfigProxy": "0x15654d11311BC099A74685fD18BAC940806C5718",
  "SystemConfig": "0x75a70C720aC2eeCE199d996Af43863665f6b89A7",
  "SystemConfigProxy": "0xB0ac0F88C41CbBF97B472AB55f17c0CE31adF140",
  "SystemOwnerSafe": "0x5cD8E9A4e956bE8d72fc449EA3a4d7401c24dEc8"
}
```

如果激活了故障证明，则需要在运行`Deploy.s.sol`部署合约时

1.针对`AnchorStateRegistry`合约，仅部署它（`deployAnchorStateRegistry()`），而不初始化它（`initializeAnchorStateRegistry()`）;

2.后续构建创世区块、启动op-geth和op-node、运行`cast rpc optimism_outputAtBlock <hex_block_number> --rpc-url http://localhost:9545`命令获取创世区块的outputRoot;

3.修改配置文件`getting-started.json`文件中的`faultGameGenesisOutputRoot`参数，设为步骤2中获取到的值，再运行`initializeAnchorStateRegistry()`

## 生成l2-allocs文件

```shell
CONTRACT_ADDRESSES_PATH=deployments/artifact.json DEPLOY_CONFIG_PATH=deploy-config/getting-started.json STATE_DUMP_PATH=deployments/l2-allocs.json forge script scripts/L2Genesis.s.sol:L2Genesis --sig 'runWithStateDump()' --chain-id $L2_CHAIN_ID
```

生成的l2-alloc文件有9.1M大小，包含L2上一些预部署合约的code字节码和合约地址。

## 构建op-geth

```shell
cd ~/op-layer2
git clone https://github.com/ethereum-optimism/op-geth.git
cd op-geth
# 最近的生产版本
# git checkout v1.101315.3
# git checkout v1.101408.0 运行报错
git checkout v1.101411.0 # make geth若报错，可将go.mod中的go 1.22修改成go 1.22.0
make geth
```

## 构建op组件

```shell
cd ~/op-layer2/optimism
git checkout op-stack/v1.9.4
make op-node op-batcher op-proposer
```

如果激活了故障证明，还需要

`make op-challenger op-program cannon cannon-prestate`

## 生成L2配置文件

```shell
# 切换到节点的生产分支（op-node、op-batcher、op-proposal）
# 基于https://github.com/ethereum-optimism/optimism/tree/v1.9.3
# git checkout op-stack/v1.9.3
git checkout op-stack/v1.9.4

# 生成L2的genesis文件
cd optimism/op-node
go run cmd/main.go genesis l2 --deploy-config ../packages/contracts-bedrock/deploy-config/getting-started.json --l1-deployments ../packages/contracts-bedrock/deployments/artifact.json --outfile.l2 genesis.json --outfile.rollup rollup.json --l1-rpc $L1_RPC_URL --l2-allocs ../packages/contracts-bedrock/deployments/l2-allocs.json

# 创建op-geth和op-node之间的身份验证
openssl rand -hex 32 > jwt.txt

cp genesis.json ~/op-layer2/op-geth
cp jwt.txt ~/op-layer2/op-geth
```

rollup.json文件示例：

```json
{
  "genesis": {
    "l1": {
      "hash": "0x3a283b2f2801ee5c283fed93312146c84918547974806823c7ea624149f56fce",
      "number": 6547525
    },
    "l2": {
      "hash": "0x98a2dc4ece56f7068c69b54192b004867cdad61b0de23bfbed708c5cbb27f0fe",
      "number": 0
    },
    "l2_time": 1724303088,
    "system_config": {
      "batcherAddr": "0x0d2897e7e3ad18df4a0571a7bacb3ffe417d3b06",
      "overhead": "0x0000000000000000000000000000000000000000000000000000000000000000",
      "scalar": "0x00000000000000000000000000000000000000000000000000000000000f4240",
      "gasLimit": 30000000
    }
  },
  "block_time": 2,
  "max_sequencer_drift": 600,
  "seq_window_size": 3600, // 3600 * 12s
  "channel_timeout": 300, 
  "channel_timeout_granite": 0,
  "l1_chain_id": 11155111,
  "l2_chain_id": 11155986,
  "regolith_time": 0,
  "canyon_time": 0,
  "delta_time": 0,
  "ecotone_time": 0,
  "fjord_time": 0,
  "batch_inbox_address": "0xff00000000000000000000000000000011155986",
  "deposit_contract_address": "0x3a4b63b8ec1596f88f7318b9ce6ce70ba10d70eb",
  "l1_system_config_address": "0xb0ac0f88c41cbbf97b472ab55f17c0ce31adf140",
  "protocol_versions_address": "0x0000000000000000000000000000000000000000",
  "da_challenge_contract_address": "0x0000000000000000000000000000000000000000"
}
```

字段解释：

```shell
"max_sequencer_drift": How far the L2 timestamp can differ from the actual L1 timestamp.
"seq_window_size": Maximum number of L1 blocks that a Sequencer can wait to incorporate the information in a specific L1 block. For example, if the window is 10 then the information in L1 block n must be incorporated by L1 block n+10.
"channel_timeout": Maximum number of L1 blocks that a transaction channel frame can be considered valid. A transaction channel frame is a chunk of a compressed batch of transactions. After the timeout, the frame is dropped.
```

通俗讲：`seq_window_size`表示L2中epoch n（也就是L1 block n）的相关数据必须在`seq_window_size`之前上传到L1。比如epoch 66100时，L2生成了区块908111-908116，`seq_window_size`设置为300，那么包含区块908111-908116交易数据的信息（也被称为data blob）必须在66100+300=66400区块前提交到L1。

其中`genesis.l1.number`是从System合约中通过调用`startBlock`函数获取的，部署System合约时，会初始化该值为当前l1的区块号。

L2的`op-node`节点启动时，将从L1的该区块开始同步。大约1s同步一个区块，如果启动`op-node`时距离部署合约时间较久，那么需要同步的区块就会比较多，同步时间长。

> 以下内容非必须不要尝试：
> 
> 经过验证，发现可以人为修改`rollup.json`中的`l1.hash`和`l1.number`为最近的区块，从而减少l2需要同步的时间。可以通过etherscan链浏览器寻找最近已经最终确定的区块（约20分钟前的区块），选定一个区块号，在终端运行该命令：
> 
> ```shell
> cast block 6497179 -r https://sepolia.infura.io/v3/1b65bb7a429c4970b5f3381ede10f8e7
> ```
> 
> 可以看到该区块的区块号、区块哈希以及时间戳。相应修改`rollup.json`文件中的`l1.number`、`l1.hash`、`l2_time`(l1该区块的时间戳)；并且需要修改`genesis.json`中的`timestamp`为`l2_time`，该值用的是16进制格式，可以在终端运行`cast to-hex 1723619988`得到时间戳的16进制表示。

## 初始化op-geth

```shell
cd ~/op-layer2/op-geth

mkdir datadir
# build/bin/geth init --datadir=datadir genesis.json # 1013xx版本
build/bin/geth init --datadir=datadir --state.scheme=hash genesis.json
```

## 启动op-geth

```shell
cd ~/op-layer2/op-geth

vim start.sh
# 输入如下内容
./build/bin/geth \
  --datadir ./datadir \
  --http \
  --http.port=8545 \
  --http.corsdomain="*" \
  --http.vhosts="*" \
  --http.addr=0.0.0.0 \
  --http.api=web3,debug,eth,txpool,net,engine \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=8546 \
  --ws.origins="*" \
  --ws.api=debug,eth,txpool,net,engine \
  --syncmode=full \
  --gcmode=archive \
  --nodiscover \
  --maxpeers=0 \
  --networkid=11155986 \
  --authrpc.vhosts="*" \
  --authrpc.addr=0.0.0.0 \
  --authrpc.port=8551 \
  --authrpc.jwtsecret=./jwt.txt \
  --rollup.disabletxpoolgossip=true \
  --verbosity=3

# 执行命令，后台启动
chmod +x start.sh
nohup ./start.sh > out.log 2>&1 &
```

**关停Rollup时**，节点关闭顺序应与启动顺序相反。

**关停op-geth**时需要注意，如果geth意外停止，数据库可能会损坏，重新启动时会导致节点出现问题。正常应使用`CTRL-C`停止在终端运行的geth，当geth在后台运行时，使用`kill -INT pid`，**也就是`kill -2 pid`。**

注意！！！一定要一对一运行op-geth和op-node，不要在一个op-node情况下运行多个op-geth，反之亦然。

## 启动op-node

```shell
cd ~/op-layer2/optimism/op-node
vim start.sh
# 输入如下内容，注意更换l1和l1.rpckind
# 如果使用blobs，还需要设置--l1.beacon=<http endpoint of L1 Beacon-node>
./bin/op-node \
  --l2=http://localhost:8551 \
  --l2.jwt-secret=./jwt.txt \
  --sequencer.enabled \
  --rollup.config=./rollup.json \
  --rpc.addr=0.0.0.0 \
  --rpc.port=8547 \
  --p2p.no-discovery \
  --rpc.enable-admin \
  --p2p.sequencer.key=$GS_SEQUENCER_PRIVATE_KEY \
  --l1=xxx \
  --l1.rpckind=standard \
  --altda.da-server=http://127.0.0.1:8082 \
  --altda.enabled=true \
  --altda.da-service=true \
  --altda.verify-on-read=false \
  --l1.beacon.ignore=true \
  --log.level=debug \
  --l1.trustrpc=true

# 执行命令，后台启动
chmod +x start.sh
nohup ./start.sh > out.log 2>&1 &
```

其中altda.da-server的值为Meeda轻节点的HTTP地址。

通过`tail out.log |grep 'Advancing bq origin'`可以查看到L2同步L1区块的最新情况。

或者运行命令查看同步情况：

```shell
cast rpc optimism_syncStatus --rpc-url http://localhost:8547|jq
```

```shell
# 查看当前区块
cast block-number --rpc-url http://localhost:8545
#查看当前safe区块
cast block-number --rpc-url http://localhost:8545 safe
# 查看当前finalized区块
cast block-number --rpc-url http://localhost:8545 finalized
# 查看区块号间隔
expr 90057 - 65711
```

查看链状态：

```shell
cast rpc optimism_outputAtBlock <hex_block_number> --rpc-url http://localhost:8547|jq
# 区块号转16进制
cast to-hex 23
```

> 当启动op-node时，op-node会从L1同步区块，由于使用的L1 RPC URL为从infura中申请的，其rpc服务受到速率限制，因此，op-node会报错`t=2024-08-09T07:10:02+0000 lvl=warn msg="Engine temporary error" err="temp: failed to fetch receipts of L1 block 0x2c8318f813b5c8082d64ba943fc030895c13d7515059d247bba2d83122c80dfe:6465262 (parent: 0xac88e63fd89bf9ba79101745f1d882783e8c096462a3cdacfc49c4a92b9274cd:6465261) for L1 sysCfg update: failed batch-retrieval: 429 Too Many Requests: {\"jsonrpc\":\"2.0\",\"error\":{\"code\":-32005,\"message\":\"batch item count exceeded\"}}"`。可以通过`l1.rpc-max-batch-size`参数微调batch size，将该值调小。
> 
> 这里我将`l1.rpc-max-batch-size`设置为10（默认为20）仍然报错。
> 
> 于是更换了一个不受限的L1 rpc url（从chainlist中挑选的一个rpc url），之后再运行op-node，结果正常同步区块。rpc url如下：
> 
> ```shell
> --l1=wss://ethereum-sepolia-rpc.publicnode.com \
> --l1.rpckind=standard \
> ```

**关停**op-node，先执行`cast rpc admin_stopSequencer --rpc-url http://localhost:8547`，再关闭op-node的进程。

## 启动op-batcher

保证op-batcher地址至少1个sepolia eth。

注意，只有在 `op-node` 完全同步完之后，再启动 `op-batcher` 。否则会出现因没完全同步导致 `batcher` 时间戳过期而频繁打包交易上链的问题。

如何确定 `op-node` 基本同步完成（即可以开始启动 `op-bathcer` ）：

查看 `op-node` 的日志内打印的同步高度的时间戳，或者进入 `op-geth` 查看 `eth.getBlockByNumber(eth.blockNumber)` 最近的L2区块的信息，找到时间戳一栏。通过对比时间戳和当前时间的时间戳，在 5000 秒内可以认为基本同步完成。

```shell
cd ~/op-layer2/optimism/op-batcher
vim start.sh
# 输入如下内容
./bin/op-batcher \
  --l2-eth-rpc=http://localhost:8545 \
  --rollup-rpc=http://localhost:8547 \
  --poll-interval=6s \
  --sub-safety-margin=6 \
  --num-confirmations=1 \
  --rpc.addr=0.0.0.0 \
  --rpc.port=8548 \
  --rpc.enable-admin \
  --max-channel-duration=0 \
  --max-l1-tx-size-bytes=30000 \
  --resubmission-timeout=48s \
  --max-pending-tx=1 \
  --metrics.enabled=true \
  --metrics.port=7301 \
  --l1-eth-rpc=xxx \
  --private-key=$GS_BATCHER_PRIVATE_KEY \
  --altda.da-server=http://127.0.0.1:8082 \
  --altda.enabled=true \
  --altda.verify-on-read=false \
  --altda.da-service=true

# 执行命令，后台启动
chmod +x start.sh
nohup ./start.sh > out.log 2>&1 &
```

max-channel-duration表示间隔多少个L1区块往L1提交数据，如果中间数据填满的话，会直接提交，然后等待1500（max-channel-duration=1500）个区块（1500 * 12s = 5h）后还会再次提交。如果该值设为0，则会根据网络配置文件getting-started.json中的sequencerWindowSize值决定，该值单位也是多少个L1区块，默认是3600，12h，该值表示op-batcher通道channel的窗口有效期。如果在5h以内，batch数据达到`--max-l1-tx-size-bytes=1040000`，也就是约0.99MB时，也将会往L1提交数据，这里用的Meeda作为DA，因此是往Meeda处上传数据。

> 然而运行后发现L2经常回滚，怀疑是batcher往L1处提交数据间隔太久，因此更改`--max-l1-tx-size-bytes=256000`.

**关停**op-batcher使用以下命令：

```shell
curl -d '{"id":0,"jsonrpc":"2.0","method":"admin_stopBatcher","params":[]}' \
    -H "Content-Type: application/json" http://localhost:8548 | jq
```

等到在日志输出中看到`Batch Submitter stopped`时，再停止该进程。

如果发送命令停止op-batcher，但还没有停止进程时，可以使用以下命令重新启动op-batcher：

```shell
curl -d '{"id":0,"jsonrpc":"2.0","method":"admin_startBatcher","params":[]}' \
    -H "Content-Type: application/json" http://localhost:8548 | jq   
```

metrics默认监听端口为7300，与op-node冲突，因此改为7301.

目前Meeda支持最大数据大小为127B * (32768 / 4) = 1016KB大小，因此`max-l1-tx-size-bytes`只能设置为1040384或更低，否则会上传失败。

## 启动op-proposer

```shell
cd ~/op-layer2/optimism/op-proposer
vim start.sh
# 输入如下内容
./bin/op-proposer \
  --poll-interval=12s \
  --rpc.port=8560 \
  --rollup-rpc=http://localhost:8547 \
  --l2oo-address=(L2OutputOracleProxy) \ #填入L2OutputOracleProxy地址
  --private-key=$GS_PROPOSER_PRIVATE_KEY \
  --l1-eth-rpc=xxx \
  --log.level=DEBUG \
  --allow-non-finalized=true

# 执行命令，后台启动
chmod +x start.sh
nohup ./start.sh > out.log 2>&1 &
```

op-proposer会间隔`--poll-interval=12s`轮询一次L2区块，并且间隔`"l2OutputOracleSubmissionInterval": 120`时间（也就是120个L2区块，4分钟）往`L2OutputOracleProxy`合约发送一笔`proposeL2Output`交易。

通过etherscan（https://sepolia.etherscan.io/address/0x372D2aE7950BDdE3Fdb5bEc57aBc470A6800f50d#readProxyContract）查看`L2OutputOracleProxy`合约信息，可以发现当前提交到哪一个L2区块以及下一个要提交output root的区块号。

如果激活了故障证明，那么在start.sh中应该删掉`--l2oo-address`，加上：

```sh
--proposal-interval=1200s \
--game-factory-address=(cat ../packages/contracts-bedrock/deployments/artifact.json | jq -r .DisputeGameFactoryProxy) \
```

**关停**op-proposer时，直接kill掉op-proposer的进程即可。关停的顺序和启动节点的顺序相反。

> 如果激活了故障证明，则还应该启动op-challenger，示例启动脚本：
> 
> ```shell
> cd ~/op-layer2/optimism/op-challenger
> vim start.sh
> # 输入如下内容
> ./bin/op-challenger \
>         --trace-type cannon \
>         --l1-eth-rpc=http://10.10.100.51:8545 \
>         --l1-beacon=http://10.10.100.51:8545 \
>         --l2-eth-rpc=http://localhost:8545 \
>         --rollup-rpc=http://localhost:8547 \
>         --selective-claim-resolution \
>         --private-key=$GS_ADMIN_PRIVATE_KEY \
>         --game-factory-address=0xee2Ab3a7812d8a6665c8b1E72ed75C6c0aAf3beF \
>         --datadir=./data/ \
>         --cannon-rollup-config=../op-node/rollup.json \
>         --cannon-l2-genesis=../op-node/genesis.json \
>         --cannon-server=../op-program/bin/op-program \
>         --cannon-bin=../cannon/bin/cannon \
>         --cannon-prestate=./op-program/bin/prestate.json
> 
> # 执行命令，后台启动
> chmod +x start.sh
> nohup ./start.sh > out.log 2>&1 &
> ```

## 相关工具

查询L1余额

```shell
 cast balance -r https://sepolia.infura.io/v3/1b65bb7a429c4970b5f3381ede10f8e7 0x57b9c7853646F991Fd8A441b252a70bc001ae10A
```

查询合约地址

```shell
cat ../packages/contracts-bedrock/deployments/artifact.json | jq -r .L2OutputOracleProxy
```

转账

```shell
cast send 0x0d2897e7e3ad18df4a0571a7bacb3ffe417d3b06 --value 1gwei --rpc-url http://localhost:8545 --private-key 0x
```

检查账户L2余额

```shell
cast balance -r http://localhost:8545 0x0d2897e7e3ad18df4a0571a7bacb3ffe417d3b06
```

## 质押

L2上账户的eth只能从L1上转移。将账户余额从L1上转移一小部分到L2上，这就叫做deposit。

```shell
cd ~/optimism/packages/contracts-bedrock
# 查看L1StandardBridgeProxy合约地址
cat deployments/artifact.json | jq -r .L1StandardBridgeProxy
# 发送一笔L1交易，将余额转到L1StandardBridgeProxy合约地址上
cast send 0x532cBdbB5EB75aABb5a161939C5DEb96AdE11664 --value 1gwei --rpc-url https://sepolia.infura.io/v3/1b65bb7a429c4970b5f3381ede10f8e7 --private-key PK
```

之后余额会真正保存到`OptimismPortalProxy`合约中。

```shell
cat deployments/artifact.json | jq -r .OptimismPortalProxy
cast balance -r https://sepolia.infura.io/v3/1b65bb7a429c4970b5f3381ede10f8e7 0xDff5D1EC955FD25ad050DE84A8e1FCbEBd771897
```

这时，L1上账户余额应已经减少。经过大约1分钟的时间，L2上账户余额会增加。

检查L2上账户的余额：

```shell
cast balance -r http://localhost:8545 0x57b9c7853646F991Fd8A441b252a70bc001ae10A
```

检查L1上账户的余额：

```shell
cast balance -r https://sepolia.infura.io/v3/41c82b2df53e455f9d38171972103fd8 0x57b9c7853646F991Fd8A441b252a70bc001ae10A
```

## 取款

将L2上账户的余额eth取回L1上账户的余额中，这一过程叫做withdrawal。

取款流程需要发送三笔交易：

### withdrawal initiating 交易

用户在L2上调用`L2CrossDomainMessenger`合约的`sendMessage`函数。该函数有三个参数：target（L1上的目标地址，转账给该账户）、message（L1转账交易的calldata）、minGasLimit（withdrawal finalizing交易的最小gas量）。

这些参数的构建比较麻烦，因此可以使用optimism sdk进行构造，普通用户可以直接通过bridge的浏览器界面点击withdrawal进行取款操作。

之后，用户需要等待op-proposer将包含这笔`withdrawal initiating transaction`的outputRoot提交至L1。

当op-proposer提出一个新的output root时，这个output proposal就包含这笔取款交易的输出根，这个输出根承诺L2上合约中`sendMessage`映射的状态，也就是承诺`withdrawal initiating transaction`，可以用来证明L2中存在待处理的提款。

在use-faultProof为false（也就是不启用故障证明）时，op-proposer会定期将L2的output root提交至L1链上的`L2OutputOracle`合约，间隔时间由`getting-started.json`文件中的`l2OutputOracleSubmissionInterval`参数决定，默认是120，也就是120 * 2s(l2-block-time) / 60s = 4min。这个值会写入`L2OutputOracle`合约中的`submissionInterval`变量中，通过etherscan区块浏览器查询SUBMISSION_INTERVAL可以看到实际的值。

在use-faultProof为true（也就是启用故障证明）时，op-proposer会定期将L2的output root提交至L1链上的`DisputeGameFactory`合约，间隔时间由`--proposal-interval`参数决定。

### withdrawal proving 交易

一旦包含初始化取款事件（`MessagePassed` event）的output root发布到L1，接下来就要证明L2上确实存在初始化取款操作。

链下步骤包含：

1.用户先从L2上获取取款交易的calldata参数，包含target、nonce、value等值；

2.再找出withdrawal initiating时，op-proposer往L1上存放outputRoot的索引；

3.然后再从L2获取证明信息，包含最近L2区块的状态根和区块哈希、取款人的L2链的状态根、取款操作的链存储证明等。

（PS：取款事件的哈希值（也就是withdrawalHash）的哈希值就是该条消息哈希值的storage slot。此外，调用rpc方法`eth_getProof`可以获取链的状态证明，需传入参数账户地址、事件slot和区块号）。

4.根据这些信息调用L1链上合约`OptimismPortal`的`proveWithdrawalTransaction()`方法。

链上步骤包含：

1.`OptimismPortal.proveWithdrawalTransaction()`运行一些健全性检查；

2.验证L2上的`L2ToL1MessagePasser.sentMessages`中是否已经存在用于提款的交易哈希值，并且该证明之前尚未提交至L1；

3.如果一切正常，就会把outputRoot、时间戳和L2 output index记录到`provenWithdrawals`映射中并发出事件；

接下来就是等到最终有效期（由getting-started.json文件中`finalizationPeriodSeconds`指定）之后，再进行withdrawal finalizing交易。

注意：当启用了故障证明时，使用的是`OptimismPortal2`。具体流程将变为：运行检查，将proof记录到`faultDisputeGame`合约中，可能会被挑战或攻击，正常节点响应挑战和防御，故障证明有效期之后，由op-challenger调用`resolveClaim()`和`resolve()`解决故障证明，声明该proof是否有效。

### withdrawal finalizing 交易

一旦故障挑战期结束，用户就可以在L1上执行和最终确定withdrawal。

## 使用Optimism SDK

使用op sdk可以方便地质押和提取以及发送交易。提取是指把eth从L2账户中转移到L1账户。

```shell
cd ~/op-layer2/optimism/packages/contracts-bedrock
# 查看合约地址
cat deployments/artifact.json
```

创建一个demo项目从而使用optimism sdk：

```shell
mkdir op-sample-project
cd op-sample-project
```

```shell
pnpm init
```

```shell
pnpm add @eth-optimism/sdk
# pnpm add viem
```

```shell
pnpm add ethers@^5
```

设置私钥环境变量

```shell
export PRIVATE_KEY=
```

```shell
node
```

```shell
const optimism = require("@eth-optimism/sdk")
# const optimism = require("viem")
const ethers = require("ethers")

const l1Provider = new ethers.providers.StaticJsonRpcProvider("https://sepolia.infura.io/v3/41c82b2df53e455f9d38171972103fd8")
const l2Provider = new ethers.providers.StaticJsonRpcProvider("http://localhost:8545")

// 注意：应该输入相应合约的代理合约地址，不然后面执行交易会报错“无法estimateGas”
const AddressManager = ''
const L1CrossDomainMessenger = ''
const L1StandardBridge = '' 
const OptimismPortal = ''
const L2OutputOracle = ''
// 如果激活故障证明，则还需指定下述合约地址
const OptimismPortal2 = ''
const DisputeGameFactory = ''

const l1ChainId = await l1Provider.getNetwork().then(network => network.chainId)
const l2ChainId = await l2Provider.getNetwork().then(network => network.chainId)

const privateKey = process.env.PRIVATE_KEY
const l1Wallet = new ethers.Wallet(privateKey, l1Provider)
const l2Wallet = new ethers.Wallet(privateKey, l2Provider)
```

现在可以从op sdk创建`CrossChainMessenger`对象实例，从而轻松处理L1和L2之间的跨域信息。

```shell
const messenger = new optimism.CrossChainMessenger({
  l1SignerOrProvider: l1Wallet,
  l2SignerOrProvider: l2Wallet,
  l1ChainId,
  l2ChainId,

  // This is the only part that differs from natively included chains.
  contracts: {
    l1: {
      AddressManager,
      L1CrossDomainMessenger,
      L1StandardBridge,
      OptimismPortal,
      L2OutputOracle,

      // Need to be set to zero for this version of the SDK.
      StateCommitmentChain: ethers.constants.AddressZero,
      CanonicalTransactionChain: ethers.constants.AddressZero,
      BondManager: ethers.constants.AddressZero,

      OptimismPortal2,
      DisputeGameFactory,
    }
  }
})
```

**查询余额：**

```shell
console.log((await l1Wallet.getBalance()).toString())
console.log((await l2Wallet.getBalance()).toString())
```

**存入eth：**

```shell
tx = await messenger.depositETH(ethers.utils.parseEther('0.001'),{gasLimit:200000})
await tx.wait()
await messenger.waitForMessageStatus(tx.hash, optimism.MessageStatus.RELAYED)
```

**提取eth：**

在L2上发起withdraw

```shell
const withdrawal = await messenger.withdrawETH(ethers.utils.parseEther('0.000716224933581508'),{gasLimit:200000})
await withdrawal.wait()
```

此时，L2上余额已经减少，但L1上钱包余额未变。需要等待withdrawal准备好向L1上的桥合约进行证明。

```shell
await messenger.waitForMessageStatus(withdrawal.hash, optimism.MessageStatus.READY_TO_PROVE)
```

向L1发送交易用来证明L2上的withdrawal交易。

```shell
await messenger.proveMessage(withdrawal.hash)
```

withdrawal的最后一步就是在L1上对withdrawal进行中继，这将在故障证明期后才会发生。本测试链设置的是12秒。

```shell
await messenger.waitForMessageStatus(withdrawal.hash, optimism.MessageStatus.READY_FOR_RELAY)
```

发起中继。

```shell
await messenger.finalizeMessage(withdrawal.hash)
```

```shell
await messenger.waitForMessageStatus(withdrawal.hash, optimism.MessageStatus.RELAYED)
```

全部流程示例：

```js
// 引入 Optimism SDK 和 ethers.js
const optimism = require('@eth-optimism/sdk');
const ethers = require('ethers');

// L1 和 L2 RPC 提供者（L1 使用 Infura，L2 是本地节点）
const l1Provider = new ethers.providers.StaticJsonRpcProvider('xxx');
const l2Provider = new ethers.providers.StaticJsonRpcProvider('http://localhost:8545');

// 合约地址，确保使用的是你网络中的有效地址
const AddressManager = '0x4909Be147744840a682b61d70D16c75F27D62D6E'
const L1CrossDomainMessenger = '0x8F7D33CEB5eF303aC1A55b371bE33bE734Ef2C20'
const L1StandardBridge = '0x0980bE2A855Ee9a3BCB4c4F31792E5d73E94EaBD'
const OptimismPortal = '0x723f7e6E724D7Ce6A5305Eb436051C85dDA0A9dE'
const L2OutputOracle = '0xc397b74290D36e29940Cd6e77A47a7067D92e2Df'
const OptimismPortal2 = ''
const DisputeGameFactory = ''

// 获取 L1 和 L2 的 Chain ID
async function getChainIds() {
    const l1ChainId = await l1Provider.getNetwork().then(network => network.chainId);
    const l2ChainId = await l2Provider.getNetwork().then(network => network.chainId);
    return { l1ChainId, l2ChainId };
}

// 私钥从环境变量中获取，确保你已经设置了 PRIVATE_KEY
const privateKey = process.env.PRIVATE_KEY;
if (!privateKey) {
    console.error('请设置 PRIVATE_KEY 环境变量！');
    process.exit(1);
}

// 创建 L1 和 L2 钱包
const l1Wallet = new ethers.Wallet(privateKey, l1Provider);
const l2Wallet = new ethers.Wallet(privateKey, l2Provider);

// 跨链信息处理器 CrossChainMessenger
async function createMessenger() {
    const { l1ChainId, l2ChainId } = await getChainIds();

    const messenger = new optimism.CrossChainMessenger({
        l1SignerOrProvider: l1Wallet,
        l2SignerOrProvider: l2Wallet,
        l1ChainId,
        l2ChainId,
        contracts: {
            l1: {
                AddressManager,
                L1CrossDomainMessenger,
                L1StandardBridge,
                OptimismPortal,
                L2OutputOracle,
                OptimismPortal2,
                DisputeGameFactory,
                StateCommitmentChain: ethers.constants.AddressZero,
                CanonicalTransactionChain: ethers.constants.AddressZero,
                BondManager: ethers.constants.AddressZero,
            }
        }
    });
    return messenger;
}

// 查询余额
async function checkBalances() {
    const l1Balance = await l1Wallet.getBalance();
    const l2Balance = await l2Wallet.getBalance();
    console.log(`L1 余额: ${ethers.utils.formatEther(l1Balance)} ETH`);
    console.log(`L2 余额: ${ethers.utils.formatEther(l2Balance)} ETH`);
}

// 从 L1 存入 ETH 到 L2
async function depositETH(messenger) {
    console.log('开始存入 ETH 到 L2...');
    const tx = await messenger.depositETH(ethers.utils.parseEther('0.02'), { gasLimit: 2000000 });
    await tx.wait();
    console.log('ETH 存入成功！');
    await messenger.waitForMessageStatus(tx.hash, optimism.MessageStatus.RELAYED);
    console.log('消息已经中��到 L2。');
}

// 从 L2 提取 ETH 到 L1

async function withdrawETH(messenger) {
    console.log('开始从 L2 提取 ETH 到 L1...');
    const withdrawal = await messenger.withdrawETH(ethers.utils.parseEther('0.001'), { gasLimit: 200000 });
    await withdrawal.wait();
    console.log('L2 提现交易完成，等待准备证明...');
    console.log('提现交易哈希: ', withdrawal.hash); // 打印提现交易的哈希
    const customHash = withdrawal.hash;
//    console.log('自定义的提现交易哈希:', customHash); // 打印自定义的提现交易哈希
    console.log('等待证明提交...');
    try {
        await messenger.waitForMessageStatus(customHash, optimism.MessageStatus.READY_TO_PROVE);
    } catch (error) {
        console.error('获取消息状态时发生错误:', error);
    }

    console.log('准备证明 L2 提现交易...');
    await messenger.proveMessage(customHash);
    console.log('证明完成，等待中继...');
    await messenger.waitForMessageStatus(customHash, optimism.MessageStatus.READY_FOR_RELAY);

    console.log('中继交易...');
    await messenger.finalizeMessage(customHash);
    await messenger.waitForMessageStatus(withdrawal.hash, optimism.MessageStatus.RELAYED)
    console.log('L2 提现成功，中继完成！');
}

// 主程序
async function main() {
    try {
        const messenger = await createMessenger();
        await checkBalances();
//        await depositETH(messenger);
        await withdrawETH(messenger);
        await checkBalances();
    } catch (error) {
        console.error('执行过程中发生错误:', error);
    }
}

// 执行主程序
main();
```
