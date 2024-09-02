# lotus 部署与使用

## 环境准备

+ 关闭swap

```shell
# 机器重启后需要再次执行
sudo swapoff /swap.img
sudo sysctl vm.swappiness=5
```

+ gpu

nvidia显卡安装

```shell
sudo apt install ubuntu-drivers-common
sudo ubuntu-drivers autoinstall

# reboot后生效，可以查看显卡信息
nvidia-smi
```

+ docker 

在需要docker执行的时候，按照如下安装

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"

sudo apt update

sudo apt install -y docker-ce

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo apt-get install -y nvidia-container-runtime

sudo usermod -aG docker mcloud

sudo service docker restart
```

logout and login again

```shell
docker pull nvidia/opencl:runtime-ubuntu18.04
```

## 运行

```
sudo apt install -y mesa-opencl-icd ocl-icd-opencl-dev
```

## 编译

如果需要编译

+ 安装依赖包

```shell
# on ubuntu 18.04
# install dependency
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common mesa-opencl-icd ocl-icd-opencl-dev gcc git bzr jq
```

+  安装rust

```shell
# on ubuntu 18.04
# install rust; follow its steps
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

+ 安装golang

golang 1.14.7版本

+ 下载源码，编译

```shell
git clone -b <branch name> https://github.com/filecoin-project/lotus.git --depth=1 lotus
cd lotus
env RUSTFLAGS="-C target-cpu=native -g" FFI_BUILD_FROM_SOURCE=1 make clean all
```

## deploy

### miner机器

+ 环境变量

可以放在~/.profile中，重新进入生效

```shell
# download parameters from; for v28
export IPFS_GATEWAY="https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/"
# for golang and gpu bellman lock；这个目录可以不配置，配置了就要确保目录存在
export TMPDIR="/disk01/storage/"
# lotus miner repo; (LOTUS_STORAGE_PATH is removed)； 默认路径为:~/.lotusminer
export LOTUS_MINER_PATH="/disk01/lotus_storage/"
# proof相关参数放在这里100GB以上; 默认路径为:/var/tmp/filecoin-proof-parameters
export FIL_PROOFS_PARAMETER_CACHE="/disk01/v28"
# GPU type
export BELLMAN_CUSTOM_GPU="GeForce RTX 2080 Ti:4352"
# pre commit1 necessary; 默认路径为:/var/tmp/filecoin-parents
export FIL_PROOFS_MAXIMIZE_CACHING=1
export FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1
export FIL_PROOFS_USE_GPU_TREE_BUILDER=1
# rust log type
export RUST_LOG=Debug
ulimit -n 65535
```

+ 启动 daemon

用于同步链上数据

```shell
# start lotus daemon; 默认路径为:~/.lotus
nohup lotus daemon > daemon.log 2>&1 &!

# check chain sync state；全部同步完成才能启动lotus-miner
lotus sync wait

# create a new key 
lotus wallet new bls
```

+ 充值

浏览器打开https://spacerace.faucet.glif.io/; 给t3地址充钱

+ 初始化 lotus-miner

```shell
# 使用t3地址
lotus init --owner=t3... --sector-size 32GiB
```

+ modify config.toml

修改LOTUS_MINER_PATH路径下的config.toml

```toml
[API]
# 换为机器的ip
ListenAddress = "/ip4/192.168.231.206/tcp/2345/http"
RemoteListenAddress = "192.168.231.206:2345"
#  Timeout = "30s"

# 使用固定端口；例如6001；119.147.213.57改为本机器的外部ip
[Libp2p]
ListenAddresses = ["/ip4/0.0.0.0/tcp/6001", "/ip6/::/tcp/6001"]
AnnounceAddresses = ["/ip4/119.147.213.57/tcp/6001"]
...

[Storage]
ParallelFetchLimit = 100
AllowAddPiece = false
AllowPreCommit1 = false
AllowPreCommit2 = false
AllowCommit = false
AllowUnseal = false
```

+ start

启动miner

```shell
nohup lotus-miner run > miner.log 2>&1 &!

# 查看信息
lotus-miner info

# 生成token用于woker连接
lotus-miner auth create-token --perm admin

# 查看网络地址
lotus-miner net listen
# 地址上链；119.147.213.57改为本机器的外网ip
lotus-miner actor set-addrs /ip4/119.147.213.57/tcp/6001/
```

### lotus-woker

#### start without docker

+ pre commit phase

```shell
#!/bin/sh

# worker main repo; WORKER_PATH is unused
export LOTUS_WORKER_PATH="/disk01/worker"
# using parent cache; necessary
export FIL_PROOFS_MAXIMIZE_CACHING=1
# Using gpu to calculate column tree
export FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1
export FIL_PROOFS_USE_GPU_TREE_BUILDER=1
export RUST_LOG=Debug
ulimit -n 65535

# store token and mapi in file
#mt=`cat ~/docker/token`
#TOKEN=`echo $mt | cut -d \: -f 1`
#MAPI=`echo $mt | cut -d \: -f 2`
# change it to your miner ip address
MAPI=/ip4/192.168.231.220/tcp/2345/http
# change it to your miner tokrn 
TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.jcGaNGLeEMGknLeDfhFX5gGwUQ0Ut3-I5bF9lMpQIlQ

export MINER_API_INFO=${TOKEN}:${MAPI}

# logfiles
DATE=`date +%y-%m-%d`
mkdir -p ~/logs
logfile=~/logs/worker.log$DATE

# check worker stats
pid=`ps aux|grep 'lotus-worker'|grep -v 'grep'|awk '{print $2}'`
if [ $pid ]; then
        echo "worker is already running"
        exit
fi


rm /disk01/worker/repo.lock
# write token it worker path
echo ${TOKEN} > ${LOTUS_WORKER_PATH}/token

# get your local ip
IP=`ifconfig |grep "192.168.231"|awk '{print $2}'`
BPort=10200

# start worker
nohup lotus-worker run --precommit1=true --precommit2=true --commit=false --listen=${IP}:${BPort} >> $logfile 2>&1  &!
```

+ commit2 phase
  
```shell
#!/bin/sh

# worker main repo; WORKER_PATH is unused
export LOTUS_WORKER_PATH="/disk01/worker"
# used to store bellman gpu.lock; default is /var/tmp/
export TMPDIR="/disk01/tmp"
# not using parent cache
export FIL_PROOFS_MAXIMIZE_CACHING=0
export RUST_LOG=Debug
ulimit -n 65535

# store token and mapi in file
#mt=`cat ~/docker/token`
#TOKEN=`echo $mt | cut -d \: -f 1`
#MAPI=`echo $mt | cut -d \: -f 2`
# change it to your miner ip address
MAPI=/ip4/192.168.231.220/tcp/2345/http
# change it to your miner tokrn 
TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.jcGaNGLeEMGknLeDfhFX5gGwUQ0Ut3-I5bF9lMpQIlQ

export MINER_API_INFO=${TOKEN}:${MAPI}

# logfiles
DATE=`date +%y-%m-%d`
mkdir -p ~/logs
logfile=~/logs/worker.log$DATE

# check worker stats
pid=`ps aux|grep 'lotus-worker'|grep -v 'grep'|awk '{print $2}'`
if [ $pid ]; then
        echo "worker is already running"
        exit
fi


rm /disk01/worker/repo.lock
# write token it worker path
echo ${TOKEN} > ${LOTUS_WORKER_PATH}/token

# get your local ip
IP=`ifconfig |grep "192.168.231"|awk '{print $2}'`
BPort=10200

# using GPU; type and its cores
export BELLMAN_CUSTOM_GPU="GeForce RTX 2080 Ti:4352" 
# not use gpu
# export BELLMAN_NO_GPU=1

# start worker
nohup lotus-worker run --addpiece=false --precommit1=false --precommit2=false --commit=true --listen=${IP}:${BPort} >> $logfile 2>&1  &!

```

#### start with docker


+ pre commit worker

```Dockerfile
FROM nvidia/opencl:runtime-ubuntu18.04

WORKDIR /app
COPY lotus-worker /app/
COPY start.sh /app/start

ENV IPADDR="" BELLMAN_CUSTOM_GPU="" FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1 FIL_PROOFS_USE_GPU_TREE_BUILDER=1 FIL_PROOFS_MAXIMIZE_CACHING=1 RUST_LOG=Debug MINER_API_INFO="" LOTUS_WORKER_PATH="/worker" TMPDIR="/tmp"

RUN mkdir -p /worker

ENTRYPOINT ["sh","-c","/app/start"]
CMD ["/bin/sh"]
```

```shell
#!/bin/bash

DATE=`date +%y-%m-%d`
logfile=$LOTUS_WORKER_PATH/log.$DATE

/app/lotus-worker run --precommit1=true --precommit2=true --commit=false --address=$IPADDR >> $logfile 2>&1
```

```shell
#!/bin/sh

# clean up containers
docker container prune --force

# Dokcerfile path
cd ~/docker/build
cp ~/lotus-worker .
# make sure this directory has lotus-worker and start.sh
docker build . -t lotus-worker:p1 --no-cache
cd ~

# parameters dir, needed by commit phase2
# local ip address of this machine
IP=`ifconfig |grep "192.168.231"|awk '{print $2}'`
BPort=13400

# MINER_API_INFO
mt=`cat ~/docker/token`
TOKEN=`echo $mt | cut -d \: -f 1`
MAPI=`echo $mt | cut -d \: -f 2`
echo ${TOKEN}
echo ${MAPI}

# share gpu

# CPU info, numa0 and numa1
NCPU=("0-15,32-47" "16-31,48-63")
for i in `seq 0 1`
do
        ipa=$IP":"$(expr $BPort + $i)
        cs=${NCPU[i]}
        gpu=$GPUINFO
        echo $ipa $cs $gpu

        IWDIR=`mount|grep worker$i|awk {'print $3'}`
        echo "worker dir is: " $IWDIR
        if [ ! -d "$IWDIR" ]; then
            echo $IWDIR "is not exists"
            exit
        fi

        docker run --privileged -d --rm --gpus device=0 --cpuset-cpus=${cs} --cpuset-mems=$i -v /var/tmp:/var/tmp -v ${IWDIR}:/worker -v /etc/localtime:/etc/localtime --network host -e FIL_PROOFS_MAXIMIZE_CACHING=1 -e IPADDR=${ipa} -e MINER_API_INFO=${TOKEN}:${MAPI} --name worker$i  lotus-worker:p1
done
```

+ commit2 worker 

```Dockerfile
FROM nvidia/opencl:runtime-ubuntu18.04

WORKDIR /app
COPY lotus-worker /app/
COPY start.sh /app/start

ENV IPADDR="" BELLMAN_CUSTOM_GPU="" RUST_LOG=Debug MINER_API_INFO="" FIL_PROOFS_PARAMETER_CACHE="/proof" LOTUS_WORKER_PATH="/worker"

RUN mkdir -p /proof /worker
expose $USEPORT

ENTRYPOINT ["sh","-c","/app/start"]
CMD ["/bin/sh"]
```

```shell
#!/bin/bash

DATE=`date +%y-%m-%d`
logfile=$LOTUS_WORKER_PATH/log.$DATE

/app/lotus-worker run --addpiece=false --precommit1=false --precommit2=false --commit=true --listen=$IPADDR >> $logfile 2>&1
```

+ start c2 using gpu

```shell
#!/bin/sh

# clean up containers
docker container prune --force

# Dokcerfile path
cd ~/docker/build
cp ~/lotus-worker .
# make sure this directory has lotus-worker and start.sh
docker build . -t lotus-worker:c2 --no-cache
cd ~

# worker parent path; 
PWDIR=/disk01
# parameters path
PADIR=/disk01/fil_proofs_cache_v27

# parameters dir, needed by commit phase2
# local ip address of this machine
IP=`ifconfig |grep "192.168.231"|awk '{print $2}'`
BPort=13400

# MINER_API_INFO
mt=`cat ~/docker/token`
TOKEN=`echo $mt | cut -d \: -f 1`
MAPI=`echo $mt | cut -d \: -f 2`
echo ${TOKEN}
echo ${MAPI}

gpu="GeForce RTX 2080 Ti:4352"
for i in `seq 1 2`
do
    ipa=$IP":"$(expr $BPort + $i)
    echo $ipa

    echo ${TOKEN} > ${PWDIR}/worker$i/token
    docker run -d --rm --gpus device=0 -v  ${PADIR}:/proof -v ${PWDIR}/worker$i:/worker -v /etc/localtime:/etc/localtime --network host -e IPADDR=$IP:$BPort -e MINER_API_INFO=${TOKEN}:${MAPI} -e BELLMAN_CUSTOM_GPU=${gpu} --name worker$i  lotus-worker:c2
}
```


```shell
#!/bin/sh

# clean up containers
docker container prune --force

# Dokcerfile path
cd ~/docker/build
cp ~/lotus-worker .
# make sure this directory has lotus-worker and start.sh
docker build . -t lotus-worker:p1 --no-cache
cd ~

# worker parent path; 
PWDIR=/disk01
# parameters path
PADIR=/disk01/fil_proofs_cache_v27

# parameters dir, needed by commit phase2
# local ip address of this machine
IP=`ifconfig |grep "192.168.231"|awk '{print $2}'`
BPort=13400

# MINER_API_INFO
mt=`cat ~/docker/token`
TOKEN=`echo $mt | cut -d \: -f 1`
MAPI=`echo $mt | cut -d \: -f 2`
echo ${TOKEN}
echo ${MAPI}

NCPU=("0-15,32-47" "16-31,48-63")
for i in `seq 0 1`
do
        ipa=$IP":"$(expr $BPort + $i)
        cs=${NCPU[i]}
        echo $ipa $cs
        echo ${TOKEN} > /disk01/worker$i/token
        docker run -d --rm --cpuset-cpus=$cs -v  ${PADIR}:/public/params/ -v ${PWDIR}/worker0:/worker  -v /etc/localtime:/etc/localtime --network host -e IPADDR=${ipa} -e MINER_API_INFO=${TOKEN}:${MAPI} -e BELLMAN_NO_GPU=1 --name worker$i  lotus-worker:c2
        # docker run -d -m 128g --memory-swap=320g --memory-swappiness=5 ${PADIR}:/public/params/ -v ${PWDIR}/worker0:/worker  -v /etc/localtime:/etc/localtime --network host -e IPADDR=${ipa} -e MINER_API_INFO=${TOKEN}:${MAPI} -e BELLMAN_NO_GPU=1 --name worker$i  lotus-worker:c2
done
```

## ops

```shell
lotus-miner rewards redeem

Filter = "jq -e '.Proposal.Client == \"t1nslxql4pck5pq7hddlzym3orxlx35wkepzjkm3i\" or .Proposal.Client == \"t1stghxhdp2w53dym2nz2jtbpk6ccd4l2lxgmezlq\" or .Proposal.Client == \"t1mcr5xkgv4jdl3rnz77outn6xbmygb55vdejgbfi\" or .Proposal.Client == \"t1qiqdbbmrdalbntnuapriirduvxu5ltsc5mhy7si\" '"
```