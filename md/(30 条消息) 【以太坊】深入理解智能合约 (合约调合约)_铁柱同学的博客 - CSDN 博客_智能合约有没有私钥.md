> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/LJFPHP/article/details/81288961)

一、前言
----

      关于[智能合约](https://so.csdn.net/so/search?q=%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6&spm=1001.2101.3001.7020)的描述，大家在网上百度能查到一大堆。看来看去也能看个似懂非懂，但是稍微具体点呢，智能合约到底都能干什么，可以转账提现吗？可以合约调用合约吗？可以发布多个合约吗？

关于智能合约的疑问真的很多，只能一边开发一边总结了。以下是我最近关于智能合约的一些新理解。

二、深入理解智能合约
----------

**1、一条链上可以发布多个智能合约。**

      这些智能合约不是相互覆盖的关系，而是可以并存的关系。  
      但是在发代币和保存数据方面，最好只能有一个合约来声明。因为如果有两个合约都有发代币的内容的话，那么部署第二个发代币合约之后，会新发一个币种，造成数据混乱等问题。

      这部分，博主刚开始一直以为一条链上只能有一个智能合约，后来开发中才意识到，原来可以发布多个合约。只要我们保持发币和数据保存部分只有一份就好，其他的逻辑方面的合约可以随便发。

**2、智能合约也是个账户**

      智能合约也是个账户，没有私钥，但是可以收到别人打过来的代币，作为中转账户使用

**收款：**外部给智能合约转账为了接收 Ether，(fallback) 回退函数必须标记为 payable。  
如果没有这样的函数，合约不能通过常规 transactions 接收 Ether。

      这部分，博主这边的需求是智能合约接收用户发送的以太币，并且转换成其他币种返回给用户。刚开始根本没想到，智能合约能作为中转账号。很神奇。

**3、智能合约提现：（前提必须是合约的操作者）**

      因为智能合约没有私钥，所以不能像普通账户一样的转账操作。但是在智能合约中，如果操作者是合约发布者的话，可以通过内置的 transfer 方法来进行转账提现：

```
//这里的address指的是你要提现的账户地址
//value代表了你要提现的金额
 address.transfer(value);
```

      提现部分，刚开始连想都不敢想。没有私钥也能提现转账？transfer 前面的地址可以任意变换吗？但是实际操作测试发现，是的，一切都可以实现。

**4、智能合约的 fallback 方法**

2300gas 限制;[http://me.tryblockchain.org/blockchain-solidity-fallback.html](http://me.tryblockchain.org/blockchain-solidity-fallback.html)  
通过这个方法，我们知道了，只有通过 this.send(1); 这种方式的 fallback 才是会受到限制的  
关于 send() 和 call) 的区别;[https://ethfans.org/topics/419](https://ethfans.org/topics/419)

为什么会说道 fallback() 呢，因为我们接收以太币的函数必须是 fallback() 函数：

```
//要接收以太币，合约里面必须要有payable关键字
//当我们使用address.send(ether to send)向某个合约直接转帐时，由于这个行为没有发送任何数据，所以接收合约总是会调用fallback函数
 function () external payable {
    require(msg.sender != address(0));
    require(msg.value != 0);
    xxxxxxxxxxx
  }
```

      但是我们根据上面我给的链接可以看到，fallback() 函数为了防止外部攻击和合约漏洞，规定了此函数只能消耗 2300gas。超过 2300gas 函数就会执行失败。我们都知道，随便一个转账都要消耗 2w + 的 gas，这 2300 个 gas 怎么可能会够呢。

      后面我们在查询资料时候才发现，fallback() 函数仅对使用 send() 方式的有 2300gas 的限制，对使用 call() 方式或者其他的操作没有这样的限制。好家伙，原来 fallback() 也没有想象中那么恐怖。

**5、智能合约调用另一个智能合约（重要！！！）**

这部分才是遇到的**重中之重**，刚开始以为只是简单的传入地址就可以调用，但是在实际测试中  
，并没有那么简单。

**（1）如果两个合约写在一起**

参考链接：  
[https://ethereum.stackexchange.com/questions/9705/how-can-you-call-a-payable-function-in-another-contract-with-arguments-and-send/9722#9722](https://ethereum.stackexchange.com/questions/9705/how-can-you-call-a-payable-function-in-another-contract-with-arguments-and-send/9722#9722)

[https://ethfans.org/topics/653](https://ethfans.org/topics/653) ：直接通过传入地址调用  
大家可以参考这两个地址，想必会有个答案。

**（2）两个合约不在一起**

      就像我们平时发布需求，不可能一次都不修改合约的内容。但是合约本身的定义就是不可篡改的，那么我们只有通过多发布几个合约来实现我们的需求了。

下面是示例代码：加入我们要在 A 合约中调用 B 合约的 test 方法

```
//先引入接口，interface就是个外部合约，你给了地址才能调他的方法。接口的名字是随意的
interface test{
//这里面的方法一定要是我们调用的B合约中的test方法，参数什么的也都保持一致。
//因为是interface，所以函数必须声明为external
  function test(address _to, uint256 _value) external returns (bool);
}

//引入接口之后，开始引入我们的A合约

contract xxxxx is Pausable{
  using SafeMath for uint256;
  uint256 public weiRaised;
  //因为solidity是静态语言，所以在我们需要使用某个变量的时候，必须要先声明一下
  test test;

  function () external payable {
   xxxx
   //这里使用实例化过的B合约，直接调用它的test方法
    require(token.test(msg.xxx, xxx));
   xxxxx
  }

  constructor(address xxx, uint256 xxx, address xx) public {
   //此处的_token是B合约的地址
    token = test(_token);
  }
}
```

```
大致步骤：

1、引入接口，接口中写入B合约的方法以及参数。
2、在A合约中申明一个静态变量，实例化B合约之后使用
3、在A合约的构造函数中实例化B合约（传入B合约的地址）
4、在A合约中直接调用B合约中的test方法
```

      interface 就是个外部合约，你给了地址才能调他的方法。这里的接口通过传入地址调用。接口中的实现类要和传入地址合约的实现类完全相同。这样在传入地址之后，就能直接调用传入地址合约中的方法。

关于为什么要通过 interface 方式调用合约，大家可以参考：  
通过 solidity 的接口实现：  
[https://my.oschina.net/u/2601303/blog/1550469](https://my.oschina.net/u/2601303/blog/1550469)  
[https://zhuanlan.zhihu.com/p/34017392](https://zhuanlan.zhihu.com/p/34017392)

三、智能合约消耗 gas 计算
---------------

**1、估算智能合约消耗的 gas 部分：**

通过以下网址即可：[http://remix.ethereum.org](http://remix.ethereum.org) remix  
（1）把自己的合约拷贝进去  
（2）右边的 compile 栏目下–》detail 按钮左边有个下拉框，选择自己的合约–》点击 detail 查询  
-》搜索 GASESTIMATES，然后会看到一个数组。上面的是部署合约和执行合约消耗的 gas，下面的是具体方法消耗的 gas 数量

**2、关于合约方法消耗 gas 多少的问题**

（1）一般来讲，当我们操作区块链进行读或者写的时候，都会消耗一定量的 gas  
（2）调用方法时候消耗的 gas 对比：[https://bitshuo.com/topic/587e03c44dea36e72c1b381b](https://bitshuo.com/topic/587e03c44dea36e72c1b381b)

      以上就是这段时间对于智能合约的新理解了。果然是纸上得来终觉浅，还是自己实践起来理解的最快。这里是我平时开发记录的笔记，可能总结的不够全面，大家可以通过给出的链接来学习（PS：有些链接需要翻墙的）。个人认为，智能合约之间的相互调用最值得学习，这部分网上的资料太少了。

加油！

**end**