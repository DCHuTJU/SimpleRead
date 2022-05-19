> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/40406683)

> 理论介绍 UTXO(Unspent Transaction output), 不知道大家还记不记得，在对比特币进行架构介绍的时候，我们就花了较大的篇幅介绍UTXO. 为什么对于UTXO要这么重视呢？因为UTXO是比特币里面使用的交易模型，也就是说…

理论介绍

UTXO(Unspent Transaction output), 不知道大家还记不记得，在对比特币进行架构介绍的时候，我们就花了较大的篇幅介绍 UTXO. 为什么对于 UTXO 要这么重视呢？因为 UTXO 是比特币里面使用的交易模型，也就是说，你所有的比特币相关的交易，都涉及到 UTXO， UTXO 不仅仅在比特币交易中承担着重要的角色，它同时也保护着你我的财产安全。 所以了解 UTXO 可以帮助我们保护我们的财产安全。

  
UTXO 这么重要，那怎么理解 UTXO 呢？ 它又有什么特别之处呢？  

随着互联网的飞速发展、移动支付的广泛普及，我相信这里的人，都已经有自己的支付宝账户了。 所以为了更接地气，让大家更好的理解 UTXO，在这里我们就类比支付宝账号来解释 UTXO，在使用支付宝的时候，相信大家都会创建一个支付宝账户。这个账户由以下信息组成：

*   账号 ID
*   密码
*   其他认证信息

当我们有支付宝账户之后，我们就可以与其他人进行转账等一系列的金融交易活动。 在支付宝里面，当我们发生一次转账的时候发生了什么呢？在支付宝里面，当我们发生一次支付的时候，看似简单的一个转账动作，其实后面涉及到一系列庞大的系统，其中最重要的就是清结算的部分。我们知道，目前的金融活动都是由中心化的机构在做担保交易，所以每个账户的对账、清洁算也是由这些中心化的机构来处理的。他们要算清楚每一个账户里面的每一笔钱。

由于清洁算一般都涉及到多方，很多个系统。所以在整个过程中， 绝大多数时候，是有部分钱算不清楚的。时常会发生多钱、少钱的情况。 尽管对于金融相关系统来说，在设计之处就充分考虑了数据的强一致性，但截止目前，基于之前的技术架构还没法完全杜绝数据不一致的情况，也就是钱算错的情况。那么算错的钱怎么办呢？数据库都掌握在别人手里，你说怎么办？

以上粗旷的讲了支付宝的支付过程，那么显然支付宝是我们熟悉的账户模型，一个用户有一个或者数个账户，这些账户之间相互独立。

比特币是去中心化的设计，所以没有一个或者几个中心机构来对账、清洁算。所以它需要有自己的一套清洁算系统。而这套系统就是 UTXO 支付模型。 整个交易、结算的过程都是由 UTXO 来完成的，其完全不借助第三方，也几乎不会发生算错的情况。同时对于 UTXO 来说，其不像账户模型，账户之间是相互独立的关系。在比特币当中，维护了一个 UTXO 池，所有的交易信息都放在这个交易池当中。在比特币里面没有账户的概念，只有地址。熟悉的人都知道，比特币里面地址跟你的公、私钥紧密相关。也可以说一个地址就唯一的确定了一对公、私钥匙。 你在比特币上的每一笔交易都会跟你的地址相关。

前面我们提到在比特币里面所有的交易都放在 UTXO 池里面，那么如何判断出哪些交易是我的，我又有多少余额呢？

嗯，相信这个问题是大家最关心的问题。其实在比特币当中，你所产生的每一笔交易都是需要通过你的公私钥唯一标识过的。也就是说，通过你的公私钥是可以逆向从 UTXO 交易池里面找出跟你相关联的交易的。

既然能找出跟你相关连的所有交易，那是不是就意味着能算出的余额？

答案当然是肯定的！

**代码实战**
--------

接下来我们结合代码来解读，在比特币系统中，一个交易包含了以下信息

*   输入集
*   输出集合
*   交易 ID

通过以下的代码，我们可以构建一个基于 utxo 的交易信息

```
def new_uxto_transaction(wallet, to, amount, utxo):
 """
    :param wallet:  发送地址
    :param to: 接收地址
    :param amount: 数量
    :param utxo:  utxo 池
    :return:
    """
    inputs = []
    outputs = []


   # todo 实现逻辑细节
    pub_key_hash = hash_pub_key(wallet.pub_key)


    acc, valid_outputs = utxo.find_spendable_outputs(pub_key_hash, amount)


    if acc < amount:
    raise ValueError("Not enough money to spend！")


    # 输入
    for txid, outs in valid_outputs.items():
    for out in outs:
    input = Input(txid, out, None, wallet.pub_key)
            inputs.append(input)


    # 输出
    from_addr = wallet.get_address()


    _out = new_utxo_outputs(amount, to)
    outputs.append(_out)


    if acc > amount:
        new_out = new_utxo_outputs(acc - amount, from_addr)
        outputs.append(new_out)


    tx = Transaction("", inputs, outputs)
    tx.ID = tx.hash()
    utxo.sign_transaction(tx, wallet.private_key)

    return tx
```

由于一个交易里面包含了交易 ID、输入、输出，所以首先我们构造了交易的输入和输出。然后对交易信息计算 hash 就是交易的 ID 值。最后用私钥加密交易，这样我们就生成了一个基于 utxo 的交易信息。 这里面我们涉及了发送地址、接受地址、数量（金钱数额）以及 utxo 池。

看来代码是不是觉得 utxo 其实并不复杂？

当然，这段代码里面有个 utxo 池，那如何构造一个 utxo 池呢？

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-


"""
This Document is Created by  At 2018/7/10


utxo 支付模型是相当重要的概念。
BTC通过UTXO来处理支付的逻辑。
核心概念为，每次交易都包含有输入跟输出。 所有的输出一定refer 输入。


在使用UTXO的过程中，需要找到整条链上的所有的能够被 pubkeyhash 也就是钱包地址所签名认证的所有的输入与输出，
然后从中找到unspent UTXO
最后根据UTXO 计算balance




=================================
|   遍历所有的区块来查找utxo        |
|   效率太低，                    |
|  is there some useful skill?  |
|                               |
|            chainstate         |
=================================




交易缓存，每次发生交易的时候计算UTXO，然后将其写到database中持久化，下一次交易重新构建utxo,
通过此方法避免每次都要重新计算所有的utxo， 提高计算效率。


"""
from core.blockchain.blockchain import BlockChain
from core.transactions.output import Output, new_utxo_outputs
from crypto.hashripe import hash_pub_key
from core.transactions.input import Input
from core.transactions.transaction import Transaction


utxoBucket = "chainstate"




# utxoset represents utxo set


class UTXO(BlockChain):
 """
    UTXO 通过继承 BlockChain来实现
    """


     def __init__(self):
     super(UTXO, self).__init__()


     def find_spendable_outputs(self, pub_hash, amount):
      """
        找出可以消费的余额
        :param pub_hash:
        :param amount:
        :return:
        """
        unspent_outputs = dict()
        accumulated = 0
     # todo 直接查询utxo use pubkey to find.
     # version_1 don't storage the utxo to database. only
     # 由于不存储utxo的状态，所以每次都需要重新计算。
     # the code need to update
     utxo = self.find_utxo()
     for tix, v in utxo.items():
         unspent_outputs[tix] = []
         for out in v:
             out_obj = Output(out["value"], out["pub_key_hash"])
             if out_obj.is_locked(pub_hash) and accumulated < amount:
                 accumulated += out["value"]
                 unspent_outputs[tix].append(out)


     return accumulated, unspent_outputs


     def utxo(self, pub_key_hash):
     """


        :return:
        """
        utxos = []


        utxo = self.find_utxo()
        for tix, v in utxo.items():
             for out in v:
                out_obj = Output(out["value"], out["pub_key_hash"])
                if out_obj.is_locked(pub_key_hash):
                    utxos.append(out_obj)


        return utxos


       def get_balance(self, pub_key_hash):
       """
        获取账户余额（未交易输出）
        :return:
        """
        balance = 0
        utxos = self.utxo(pub_key_hash)
        for out in utxos:
            balance += out.value


        return balance


     def count_transaction(self):
     """
        计算交易的数量
        :return:
        """
        count = 0


        utxo = self.find_utxo()
         for k, v in utxo.items():
            count += 1


        return count
```

以上的代码就构造了 utxo 池，相信了解 python 的同学已经看出来了，我们在实现 utxo 这个类的时候，继承了 blockchain 这个类，同时很多方法也是从 blockchain 这个类里面直接拿出来用的。 我们暂且不去看 blockchain，我们看 utxo.

在 utxo 中我们实现了以下几个方法：

*   find_spendable_outputs
*   utxo
*   get_balance
*   count_transaction

以上几个方法除了最后一个方法，是计算整个 utxo 池里所有的交易数之外，其他的几个方法都是跟地址有关的。

find_spendable_outputs 是查找某地址下的可用余额，utxo 是某个地址下的 utxo 池，get_balance 是获取某地址的余额。

相信这个讲大家都很清楚。

接下来我们看看 blockchain 相关的代码，其实在之前的章节中，我们就实现了 blockchain，但是在之前，我们没有加入 utxo.

所以这里我们看看，加入了 utxo 之后的 blockchain 的代码是什么样的？

```
class BlockChain:
    """
    doc New blockchain
    """


     def __init__(self):
     # At first storage blocks in a memory list
     # and then we storage this in redis
     # self.blocks = []


     self.blocks = Redis()
     self.current_hash = None


     def add_block(self, new_block):
        """
        添加block到链上
        :param new_block:
        :return:
        """


         # there we should mark last block hash to create a new block.
         # in the redis the l is the last block hash value key.


         pow = ProofOfWork(new_block, new_block["Nonce"])
         if pow.validate():
         #self.blocks.append(new_block)
             if not self.blocks.get(new_block["Hash"]):
             # 不存在的加入
             self.blocks.set(new_block["Hash"], new_block)
             self.blocks.set("l", new_block["Hash"])


     def get_height(self):
        last_hash = self.blocks.get("l")
        last_block = self.blocks.get(last_hash).decode()


        return eval(last_block)["Height"]


     def iterator(self):


         # 这里没有迭代创世区块
        if not self.current_hash:
        self.current_hash = self.blocks.get("l")


        last_block = self.blocks.get(self.current_hash).decode()


        if eval(last_block)["PrevBlockHash"]:
        self.current_hash = eval(last_block)["PrevBlockHash"]
        yield eval(last_block)


     def get_block(self, block_hash):
         block = self.blocks.get(block_hash)
         if block:
             return eval(block.decode)


         else:
             return "区块不存在"
...
```

代码量相对之前，有些许增加，我们在之前的基础上，修改了 add_block 方法，将生成区块的代码拆出来换成了 mine_block 方法，然后增加了在链上构造 utxo 池、查找交易、获取区块高度、迭代整个链、加密、认证等一系列方法。代码里面体现的比较详细，建议大家直接看代码阅读。（码字实在有点累）

专栏地址
----

[EOS 区块链技术开发 － 小专栏](https://link.zhihu.com/?target=https%3A//xiaozhuanlan.com/eosio)

**项目地址**
--------

- [https://github.com/csunny/py-bitcoin](https://link.zhihu.com/?target=https%3A//github.com/csunny/py-bitcoin)