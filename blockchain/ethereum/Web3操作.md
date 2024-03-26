## Web3操作

```js
var Web3 = require('web3');
var Tx = require('ethereumjs-tx').Transaction;
var solc = require('solc');
var fs = require('fs');
const { time } = require('console');
const { ENGINE_METHOD_CIPHERS } = require('constants');
//init
if (typeof web3 !== 'undefined') {
    web3 = new Web3(web3.currentProvider);
} else {
    web3 = new Web3(new Web3.providers.HttpProvider("http://localhost:8545"));
}


//查询账户余额，不可以var res=web3.eth.getBalance(); console.log(res);因为异步操作
var account = '0x4a044980C2a4869E75D709E89c1eEB83Ecb68888';
web3.eth.getBalance(account).then(console.log); //promise方式
web3.eth.getBalance(account, function(error, result) { //callback方式
    if (!error) {
        console.log(result);
    } else {
        console.log(error);
    }
});


//转账，sendTransaction方法适合账户在本地有私钥的情况
var toAccount = '0x915C4f9787ba15cD29EeBba18cA6A6969A126386';
web3.eth.getBalance(toAccount).then(console.log);
var money = web3.utils.toWei('1', 'ether');
web3.eth.sendTransaction({
    from: account,
    to: toAccount,
    value: money
}).then(function(receipt) {
    console.log(receipt);
    web3.eth.getBalance(account, function(error, result) { //callback方式
        if (!error) {
            console.log("from:", result);
        } else {
            console.log(error);
        }
    });
    web3.eth.getBalance(toAccount, function(error, result) { //callback方式
        if (!error) {
            console.log("to:", result);
        } else {
            console.log(error);
        }
    });
});


//转账，sendSignedTransaction先获取私钥
var privateKey = new Buffer.from('5f24ad3c8b261caf020cecb732d10ddfdf368c2314ecd07719c71d341268c0a6', 'hex');
web3.eth.getTransactionCount(account, web3.eth.defaultBlock.pending).then(function(nonce) {
    var txData = {
        nonce: web3.utils.toHex(nonce++),
        gasPrice: web3.utils.toHex(10e9),
        gasLimit: web3.utils.toHex(99000),
        from: account,
        to: toAccount,
        value: web3.utils.toHex(money),
        data: '0x00' //此处不能为空
    }
    var tx = new Tx(txData);
    tx.sign(privateKey);
    var serializedTx = tx.serialize();
    web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex'))
        .on('receipt', function(receipt) {
            console.log(receipt);
            web3.eth.getBalance(account, function(error, result) { //callback方式
                if (!error) {
                    console.log("from:", result);
                } else {
                    console.log(error);
                }
            });
            web3.eth.getBalance(toAccount, function(error, result) { //callback方式
                if (!error) {
                    console.log("to:", result);
                } else {
                    console.log(error);
                }
            });
        });
})

//获取区块信息
web3.eth.getBlockNumber()
    .then(console.log);
web3.eth.getBlockNumber(function(error, result) {
    if (!error) {
        console.log(result);
    } else {
        console.log(error)
    }
})

//查看block.gasLimit
web3.eth.getBlock('latest', false, function(error, result) {
    if (!error) {
        console.log("block.gasLimit:", result.gasLimit);
    } else {
        console.log(error);
    }
})

//部署合约，采用构造交易的方式。如果合约构造函数需要输入参数的话，用下述方式部署合约
var bytecode = fs.readFileSync("./eth/Indexer.bin"); //从本地读取合约bytecode
var data = '0x' + bytecode;
web3.eth.getTransactionCount(account, web3.eth.defaultBlock.pending).then(function(nonce) {
        var txData = {
            nonce: web3.utils.toHex(nonce++),
            gasPrice: web3.utils.toHex(10e9),
            gasLimit: web3.utils.toHex(6721975),
            from: account,
            data: data
        }
        var tx = new Tx(txData);
        tx.sign(privateKey);
        var serializedTx = tx.serialize();
        web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex'))
            .on('receipt', function(receipt) {
                console.log(receipt);
            })
            .on('error', console.error);
    })
    //返回结果
    // {
    //     transactionHash: '0x2d82405643944eaa70b82d1f68dcf2c64a2352b1f5b47a4bddf144f17fcbd430',
    //     transactionIndex: 0,
    //     blockHash: '0x84cbe8e6290af10d479dbc216ea5f258901458a0136e55e68630b1bd188c7cd9',
    //     blockNumber: 1,
    //     from: '0x4a044980c2a4869e75d709e89c1eeb83ecb68888',
    //     to: null,
    //     gasUsed: 1161824,
    //     cumulativeGasUsed: 1161824,
    //     contractAddress: '0x2Ca8340D403c8ECDd14Af18b8aF8a7176b48e494',
    //     logs: [],
    //     status: true,
    //     logsBloom: '0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000'
    // }

//部署合约，采用mycontract.deploy()的方式。构造函数需要传参的话：myContact.deploy({arguments: ["Hello world!"]})
var indexerABI = [{ "anonymous": false, "inputs": [{ "indexed": false, "internalType": "string", "name": "key", "type": "string" }, { "indexed": false, "internalType": "address", "name": "owner", "type": "address" }, { "indexed": false, "internalType": "address", "name": "resolver", "type": "address" }], "name": "Add", "type": "event" }, { "anonymous": false, "inputs": [{ "indexed": false, "internalType": "string", "name": "key", "type": "string" }, { "indexed": false, "internalType": "address", "name": "oldOwner", "type": "address" }, { "indexed": false, "internalType": "address", "name": "newOwner", "type": "address" }], "name": "AlterOwner", "type": "event" }, { "anonymous": false, "inputs": [{ "indexed": false, "internalType": "address", "name": "from", "type": "address" }, { "indexed": false, "internalType": "address", "name": "to", "type": "address" }], "name": "AlterOwner", "type": "event" }, { "anonymous": false, "inputs": [{ "indexed": false, "internalType": "string", "name": "key", "type": "string" }, { "indexed": false, "internalType": "address", "name": "oldResolver", "type": "address" }, { "indexed": false, "internalType": "address", "name": "newResolver", "type": "address" }], "name": "AlterResolver", "type": "event" }, { "anonymous": false, "inputs": [{ "indexed": false, "internalType": "string", "name": "data", "type": "string" }], "name": "Error", "type": "event" }, { "inputs": [{ "internalType": "string", "name": "key", "type": "string" }, { "internalType": "address", "name": "resolver", "type": "address" }], "name": "add", "outputs": [{ "internalType": "bool", "name": "", "type": "bool" }], "stateMutability": "nonpayable", "type": "function" }, { "inputs": [{ "internalType": "address", "name": "newOwner", "type": "address" }], "name": "alterOwner", "outputs": [{ "internalType": "bool", "name": "", "type": "bool" }], "stateMutability": "nonpayable", "type": "function" }, { "inputs": [{ "internalType": "string", "name": "key", "type": "string" }, { "internalType": "address", "name": "owner", "type": "address" }], "name": "alterOwner", "outputs": [{ "internalType": "bool", "name": "", "type": "bool" }], "stateMutability": "nonpayable", "type": "function" }, { "inputs": [{ "internalType": "string", "name": "key", "type": "string" }, { "internalType": "address", "name": "resolver", "type": "address" }], "name": "alterResolver", "outputs": [{ "internalType": "bool", "name": "", "type": "bool" }], "stateMutability": "nonpayable", "type": "function" }, { "inputs": [{ "internalType": "string", "name": "key", "type": "string" }], "name": "get", "outputs": [{ "internalType": "address", "name": "", "type": "address" }, { "internalType": "address", "name": "", "type": "address" }], "stateMutability": "view", "type": "function" }, { "inputs": [], "name": "getOwner", "outputs": [{ "internalType": "address", "name": "", "type": "address" }], "stateMutability": "view", "type": "function" }, { "inputs": [{ "internalType": "bool", "name": "param", "type": "bool" }], "name": "setBanned", "outputs": [], "stateMutability": "nonpayable", "type": "function" }];
web3.eth.getTransactionCount(account, web3.eth.defaultBlock.pending).then(function(nonce) {
    const myContact = new web3.eth.Contract(indexerABI, { data: data })
    myContact.deploy()
        .send({ from: account, gas: web3.utils.toHex(6721975) }) // send() returns a Promise & EventEmit
        .on('transactionHash', function(transactionHash) {
            console.log("deploy transaction hash: ", transactionHash)
        })
        .on('receipt', function(receipt) {
            console.log("deploy receipt: ", receipt)
        })
        .on('confirmation', function(confirmationNum, receipt) {
            console.log("got confirmations number: ", confirmationNum)
        })
        .then(async function(myContactInstance) {
            console.log("deployed successfully.")
        })
        .catch(err => {
            console.log("Error: failed to deploy, detail:", err)
        })
})

//调用合约函数
var contractAddress = '0x2Ca8340D403c8ECDd14Af18b8aF8a7176b48e494';
var contractAddress2 = '0x471a0419Ea48EDe57940B693514A48343d575547';
//构建合约实例
var myContract = new web3.eth.Contract(indexerABI, contractAddress);

//调用'constant'函数
myContract.methods.get('testV1').call({ from: account }, function(error, result) {
    if (!error) {
        console.log("testV1:", result);
    } else {
        console.log(error);
    }
})
myContract.methods.getOwner().call({ from: account }, function(error, result) {
    if (!error) {
        console.log("owner:", result);
    } else {
        console.log(error);
    }
})

//调用‘写’函数
var resolverAddress = '0x328c20c40a14Cb6637741C1B93DAaeFE79C78bd6';
myContract.methods.add('testV1', resolverAddress).send({ from: account }, function(error, result) {
    if (!error) {
        console.log('txHash:', result);
    } else {
        console.log(error);
    }
    myContract.methods.get('testV1').call({ from: account }, function(error, result) {
        if (!error) {
            console.log("testV1:", result);
        } else {
            console.log(error);
        }
    })
})

//调用‘写’函数，通过发送交易形式
var resolverAddress2 = '0x0356b9396E9337305B18D6Fa00E70680c48CCFe1';
web3.eth.getTransactionCount(account, web3.eth.defaultBlock.pending).then(function(nonce) {
    var txData = {
        nonce: web3.utils.toHex(nonce++),
        gasPrice: web3.utils.toHex(10e9),
        gasLimit: web3.utils.toHex(6721975),
        from: account,
        to: contractAddress,
        data: myContract.methods.add('testV2', resolverAddress2).encodeABI()
    }
    var tx = new Tx(txData);
    tx.sign(privateKey);
    var serializedTx = tx.serialize();
    web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex'), function(error, result) {
        if (!error) {
            console.log('txHash:', result);
        } else {
            console.log(error);
        }
        myContract.methods.get('testV2').call({ from: account }, function(error, result) {
            if (!error) {
                console.log("testV2:", result);
            } else {
                console.log(error);
            }
        })
    })
})
```
