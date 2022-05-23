> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_36228377/article/details/124574465)

### 文章目录

*   [1 CALL 和 CALLCODE](#1_CALL__CALLCODE_15)
*   [2 CALLCODE 和 DELEGATECALL](#2_CALLCODE__DELEGATECALL_63)
*   [3 STATICCALL](#3_STATICCALL_117)
*   [4 总结](#4__132)

孔乙己懂 “回” 字的四种写法，你会智能合约的四种调用方式吗？

在中大型的项目中，我们不可能在一个[智能合约](https://so.csdn.net/so/search?q=%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6&spm=1001.2101.3001.7020)中实现所有的功能，而且这样也不利于分工合作。一般情况下，我们会把代码按功能划分到不同的库或者合约中，然后提供接口互相调用。

在 Solidity 中，如果只是为了代码复用，我们会把公共代码抽出来，部署到一个 library 中，后面就可以像调用 C 库、Java 库一样使用了。但是 library 中不允许定义任何 storage 类型的变量，这就意味着 library 不能修改合约的状态。如果需要修改合约状态，我们需要部署一个新的合约，这就涉及到合约调用合约的情况。

合约调用合约有下面 4 种方式：

*   CALL
*   CALLCODE
*   DELEGATECALL
*   STATICCALL

1 CALL 和 CALLCODE
=================

CALL 和 CALLCODE 的区别在于：代码执行的上下文环境不同。

具体来说，CALL 修改的是`被调用者`的 storage，而 CALLCODE 修改的是`调用者`的 storage。  
![](https://img-blog.csdnimg.cn/4d16d6e3cb6440d385c4eacb6cfc4fca.png)

我们写个合约验证一下我们的理解：

```
pragma solidity ^0.4.25;

contract A {
    int public x;
    

    function inc_call(address _contractAddress) public {
        _contractAddress.call(bytes4(keccak256("inc()")));
    }
    function inc_callcode(address _contractAddress) public {
        _contractAddress.callcode(bytes4(keccak256("inc()")));
    }

}

contract B {
    int public x;
    

    function inc() public {
        x++;
    }

}
```

我们先调用一下 inc_call()，然后查询合约 A 和 B 中 x 的值有什么变化：  
![](https://img-blog.csdnimg.cn/e476714da93f4a2d99ee70f1cd10c87b.png)

可以发现，合约 B 中的 x 被修改了，而合约 A 中的 x 还等于 0。

我们再调用一下 inc_callcode() 试试：  
![](https://img-blog.csdnimg.cn/c598de0dc7564f68af1af12b29333503.png)

可以发现，这次修改的是合约 A 中 x，合约 B 中的 x 保持不变。

2 CALLCODE 和 DELEGATECALL
=========================

实际上，可以认为 DELEGATECALL 是 CALLCODE 的一个 bugfix 版本，官方已经不建议使用 CALLCODE 了。

CALLCODE 和 DELEGATECALL 的区别在于：`msg.sender`不同。

具体来说，DELEGATECALL 会一直使用原始调用者的地址，而 CALLCODE 不会。  
![](https://img-blog.csdnimg.cn/33cd1fd4eb614f7b93a262c3a9c5fad1.png)

我们还是写一段代码来验证我们的理解：

```
pragma solidity ^0.4.25;

contract A {
    int public x;
    

    function inc_callcode(address _contractAddress) public {
        _contractAddress.callcode(bytes4(keccak256("inc()")));
    }
    function inc_delegatecall(address _contractAddress) public {
        _contractAddress.delegatecall(bytes4(keccak256("inc()")));
    }

}

contract B {
    int public x;
    

    event senderAddr(address);
    function inc() public {
        x++;
        emit senderAddr(msg.sender);
    }

}
```

我们首先调用一下 inc_callcode()，观察一下 log 输出：

![](https://img-blog.csdnimg.cn/20181107163125761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1R1cmtleUNvY2s=,size_16,color_FFFFFF,t_70)

可以发现，msg.sender 指向合约 A 的地址，而非交易发起者的地址。

我们再调用一下 inc_delegatecall()，观察一下 log 输出：

![](https://img-blog.csdnimg.cn/20181107163155304.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1R1cmtleUNvY2s=,size_16,color_FFFFFF,t_70)

可以发现，msg.sender 指向的是交易的发起者。

3 STATICCALL
============

STATICCALL 放在这里似乎有滥竽充数之嫌，因为目前 Solidity 中并没有一个 low level API 可以直接调用它，仅仅是计划将来在编译器层面把调用 view 和 pure 类型的函数编译成 STATICCALL 指令。

view 类型的函数表明其不能修改状态变量，而 pure 类型的函数则更加严格，连读取状态变量都不允许。

目前是在编译阶段来检查这一点的，如果不符合规定则会出现编译错误。如果将来换成 STATICCALL 指令，就可以完全在运行时阶段来保证这一点了，你可能会看到一个执行失败的交易。

话不多说，我们就先看看 STATICCALL 的实现代码吧：

![](https://img-blog.csdnimg.cn/20181107163210212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1R1cmtleUNvY2s=,size_16,color_FFFFFF,t_70)

可以看到，解释器增加了一个 readOnly 属性，STATICCALL 会把该属性置为 true，如果出现状态变量的写操作，则会返回一个 errWriteProtection 错误。

就聊到这里，相信大家已经掌握了合约的四种调用方式了吧～

4 总结
====

总结一下：

1.  `call`使用的是被调用者的上下文
2.  `callcode`和`delegatecall`使用调用者的上下文
3.  `call`可以涉及账户间的操作，另外两个可以理解为了放在以太坊上的类库，仅仅是调用他们的函数方法和 storage。
4.  `callcode`和`delegatecall` 的区别在于后者将`calleradress`和`value`始终指向原始调用的 eoa 外部账户，后者可能的最大用处就是可以在调用`delegatecall`的时候再调用`call`来对原始账户进行转账操作。