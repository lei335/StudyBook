# 优化合约Gas消耗

## 1.避免将合约用作数据存储。

*这个例子为什么？*

不好的代码实现：

```solidity
contract User {
  uint256 public amount;
  bool public isAdmin;
  function User(uint256 _amount, bool _isAdmin) {
    amount = _amount;
    isAdmin = _isAdmin;
  }
}
```

好的代码实现：

```solidity
contract MyContract {
  mapping(address => uint256) amount;
  mapping(address => bool) isAdmin;
}
```

另一种OK的代码实现：

```solidity
contract MyContract {
  struct {
    uint256 amount;
    bool isAdmin;
  }
mapping(address => User) users;
}
```

## 2.避免重复写入

最好一次在最后尽可能多地写入到存储变量。

## 3.合约的状态变量要正确的排序和分组

将数据大小分组为32个字节，并将通常同时更新的变量放在一起。

## 4.更改交易输入数据的排序

合约交易的基本气体是21000。输入数据为每字节68个GAS，如果字节为0x00则为4个GAS。

例如，如果数据为0x0dbe671f，则气体为68 * 4 = 272; 如果是0x0000001f，它是68 * 1 + 4 * 3 = 80。

由于所有参数都是32字节，因此当参数为零时，气体消耗最小。它将是32 * 4 = 128。最大值如下：

>  n * 68 +（32-n）* 4 的字节数 （n为非0字节数） 

## 5.调用外部合约

调用外部合约执行EXTCODESIZE和CALL指令。基本消耗1400 GAS。除非必要，否则不建议拆分多个合同。可以使用多个继承来管理代码。 ？？？

> 需要测试一下，使用继承和外部合约调用两种方式，哪一种更节省gas.

## 6.去掉不必要的事件

## 7.循环中多用便宜的操作

比如：

```solidity
不好的代码，例如：
 uint sum = 0;
 function p3 ( uint x ){
     for ( uint i = 0 ; i < x ; i++)
         sum += i; }
在上面的代码中，由于sum每次在循环内读取和写入存储变量，所以在每次迭代时都会发生昂贵的存储操作。这可以通过引入如下的局部变量来节省GAS来避免。

好的代码，例如：
 uint sum = 0;
 function p3 ( uint x ){
     uint temp = 0;
     for ( uint i = 0 ; i < x ; i++)
         temp += i; }
     sum += temp;
```

## 8.固定长度

尽量使用固定长度的字节数组，bytes32比string便宜。

## 9.在函数体内多使用局部变量进行操作，而不是每次读取合约状态变量

通过减少合约中状态变量操作，可以节省gas。比如：

```
uint256 _kLast = kLast; // gas savings
if (_kLast != 0) {
     uint256 rootK = Math.sqrt(uint256(_reserve0).mul(_reserve1));
     uint256 rootKLast = Math.sqrt(_kLast);
}
```

其中kLast是合约中的一个状态变量，在函数中，使用一个局部变量_kLast来记录kLast的值，后续就访问_kLast，而不是访问kLast.

## 10.获取数组信息

两种情况：

* 一次性直接获取10个数组元素；

* 每次获取1个数组元素，调用该函数10次。

这两种情况哪一种更节省Gas?

如果是外部账户获取，那么两种情况一样，都不会消耗Gas，因为是从本地矿工获取的信息。但是如果是由另外一个合约来调用该函数获取信息，那么就会消耗Gas，因为需要其他矿工的帮忙。

由另外一个合约来获取信息的情况下，第一种情况消耗的Gas要少得多，因为EVM执行的操作更少。

## 11.bytes32和bytes16[2]

在storage中，都只占用一个存储插槽，即slot，因为会有变量打包。但是在memory中，并不会进行变量打包，所以bytes32只占用一个存储插槽，而bytes16[2]会占用两个存储插槽。

具体可参考：https://foresightnews.pro/article/detail/36639