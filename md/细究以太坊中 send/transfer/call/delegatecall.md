> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/35292014)

> 以太坊中send/transfer/call都能进行转ETH的操作，但这几个关键字的不同对solidity开发者来说非常重要，容易产生安全问题。 delegatecall很容易与call混淆，这里也拿出来区分。 send原型 <address>.send(uin…

以太坊中 send/transfer/call 都能进行转 ETH 的操作，但这几个关键字的不同对 solidity 开发者来说非常重要，容易产生安全问题。

delegatecall 很容易与 call 混淆，这里也拿出来区分。

send
----

原型

<address>.send(uint256 amount) returns (bool)

简介

**向 address** 发送 amount 数量的 **Wei**（注意单位），**如果执行失败返回 false**。发送的同时**传输 2300gas**，gas 数量不可调整

transfer
--------

原型

<address>.transfer(uint256 amount)

简介

**向 address** 发送 amount 数量的 **Wei**（注意单位），**如果执行失败则 throw**。发送的同时**传输 2300gas**，gas 数量不可调整

send 与 transfer 对比简析
--------------------

相同之处

（1）均是向 address 发送 ETH（以 Wei 做单位）

（2）发送的同时传输 2300gas（gas 数量很少，只允许接收方合约执行最简单的操作）

不同之处

（1）send 执行失败返回 false，transfer 执行失败则会 throw。这也就意味着使用 send 时一定要判断是否执行成功。

推荐

（1）默认情况下最好使用 transfer（因为内置了执行失败的处理）

call
----

原型

<address>.call(...) returns (bool)

简介

以 **address（被调用合约）的身份**调用 address 内的函数，默认情况下**将所有可用的 gas 传输**过去，gas 传输量可调。执行失败时返回 false

实例

```
//call的函数调用
nameReg.call("register", "MyName");
nameReg.call(bytes4(keccak256("fun(uint256)")), a);
//设置调用时的gas和传输的钱
nameReg.call.gas(1000000).value(1 ether)("register", "MyName");
```

delegatecall
------------

原型

<address>.delegatecall(...) returns (bool)

简介

以**调用合约的身份**调用 address 内的函数，默认情况下**将所有可用的 gas 传输**过去，gas 传输量可调。执行失败时返回 false。本函数目的在于让合约能够在不传输自身状态 (如 balance、storage) 的情况下使用其他合约的代码。

实例

```
nameReg.delagatecall.gas(1000000)("register", "MyName");
//delegatecall不支持.value
```

call 与 delegatecall 对比简析
------------------------

相同之处

（1）调用时会将本合约所有可用的 gas 传输过去

（2）执行失败均返回 false

不同之处

（1）call 可以使用. value 传 ETH 给被调用合约

（2）假设在 contract_test 合约中分别有 nameReg.call("somefunction") 以及 nameReg.delegatecall("somefunction")

nameReg.call 以 nameReg 合约的身份在 nameReg 中执行 somefunction

nameReg.delegatecall 以 contract_test 合约的身份在 nameReg 中执行 somefunction

（3）delegatecall 的目的就是让合约在不用传输自身状态 (如 balance、storage) 的情况下可以使用其他合约的代码

实例

以下代码可在 [Remix](https://link.zhihu.com/?target=https%3A//remix.ethereum.org) 上操作

```
pragma solidity ^0.4.17;

contract SomeContract {
    event callMeMaybeEvent(address _from);
    function callMeMaybe() payable public {
        callMeMaybeEvent(this);
    }
}

contract ThatCallsSomeContract {
    function callTheOtherContract(address _contractAddress) public {
        require(_contractAddress.call(bytes4(keccak256("callMeMaybe()"))));
        require(_contractAddress.delegatecall(bytes4(keccak256("callMeMaybe()"))));
        SomeLib.calledSomeLibFun();
    }
}

library SomeLib {
    event calledSomeLib(address _from);
    function calledSomeLibFun() public {
        calledSomeLib(this);
    }
}
```

操作流程

（1）创建 SomeContract

（2）创建 ThatCallsSomeContract

（3）以 SomeContract 合约的地址作为参数执行 ThatCallsSomeContract 合约中的 callTheOtherContract

（4）查看执行结果中的 logs 字段

（5）查看结果会发现，_contractAddress.call 使用的是 SomeContract 的地址，_contractAddress.delegatecall 以及 SomeLib.calledSomeLibFun 使用的是 ThatCallsSomeContract 的地址

以上操作的视频演示可以看

参考资料

[Units and Globally Available Variables](https://link.zhihu.com/?target=https%3A//solidity.readthedocs.io/en/develop/units-and-global-variables.html%23address-related)

[Solidity: .call() vs. .delegatecall() vs. Libraries](https://link.zhihu.com/?target=https%3A//vomtom.at/address-call-vs-delegatecall-vs-libraries/)