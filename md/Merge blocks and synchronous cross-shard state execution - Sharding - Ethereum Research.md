> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ethresear.ch](https://ethresear.ch/t/merge-blocks-and-synchronous-cross-shard-state-execution/1240/4)

> We know that with merge blocks and clever use of fork choice rules, we can have a sharded chain where a block on one shard is atomic with a block on another shard; that is, either both blocks are part of the final chain or neither are, and within that block there can be transactions that make synchronous calls between the two shards (which are useful for, among other things, solving train-and-hotel problems). However, there remains a challenge: how do we do state execution for such blocks? That ...

![](https://ethresear.ch/user_avatar/ethresear.ch/vbuterin/45/91_2.png)

We know that with [merge blocks and clever use of fork choice rules 141](https://twitter.com/VladZamfir/status/945358660187893761), we can have a sharded chain where a block on one shard is atomic with a block on another shard; that is, either both blocks are part of the final chain or neither are, and within that block there can be transactions that make synchronous calls between the two shards (which are useful for, among other things, solving [train-and-hotel problems 62](https://github.com/ethereum/wiki/wiki/Sharding-FAQ#what-is-the-train-and-hotel-problem)). However, there remains a challenge: how do we do state execution for such blocks? That is, suppose that we have block A on shard 1 and block B on shard 2, which are merged; how does a node which is a full client on shard 1 but a light client on shard 2 execute the merged blocks, when the execution clearly depends on the pre-state of shard 1 and shard 2.

One possibility is if we operate in the stateless client model, in which case any nodes that need to execute that block could simply download the witnesses from all blocks, and execute it. In general, it’s not reasonable to expect clients to have the full state of all shards, so if we generally expect clients to hold the full state on one shard, then they will be stateless on the other shard. We can expect merge transactions to hop between the two shards many times; for example:

```
x1 = value_on_shard_1
x2 = sha3(x1, value_on_shard_2)
x3 = sha3(x2, another_value_on_shard_1)
x4 = sha3(x3, another_value_on_shard_2)
.......
```

This precludes the possibility of clients that are full on one shard and _light_ (ie. non-executing) on the other shard, as one cannot complete the execution without seeing all of the intermediate steps, and it is impractical to pause execution many times to download light client proofs for all the intermediate steps. This implies that a setup is required where all transactions in the merge block come with access lists and witnesses (as in the stateless client spec), so that clients who just see the state root for both shards can execute it.

Note that even with this setup, the specific witnesses required can only be calculated after the post-state of the direct parent of the merge block has been computed. This implies a round of network latency between computing the state at block N and computing the state at block N+1, which is an efficiency limitation.

It is also worth noting that merged state execution technically does not require merge blocks. Instead, one can simply define a deterministic function that assigns shard IDs into pairs at each block number (eg. if we want to cycle evenly, and there are `k` shards and `k` is prime, then at block number N shards, shuffle shard IDs with `i -> (i * N) % k` and then merge-execute blocks 1 and 2, 3 and 4, etc using post-shuffling IDs). Merge-executing two blocks would simply mean that a transaction starting on one block is capable of reading and writing state on the other block.

Another possible construction is to cycle through which shard is the “primary” shard during each block height, and during that height allow transactions from that shard to read and write to _all_ shards. All executors of all shards would statelessly process the transactions from that shard first, then process their own transactions.

The reason why merge blocks are not required is that state execution can be separated from consensus on data, and so if absolutely needed state executors can simply wait until blocks on all shards are finalized up to some height before calculating state up to that height (though a more optimistic approach, using conditional claims, is also possible).

![](https://ethresear.ch/user_avatar/ethresear.ch/maxc/45/675_2.png)

Would be cool to know more about how merge blocks work.

> Another possible construction is to cycle through which shard is the “primary” shard during each block height, and during that height allow transactions from that shard to read and write to all shards. All executors of all shards would statelessly process the transactions from that shard first, then process their own transactions.

If multi-shard transactions were great in number - say 1/2 of all transactions but there were only 200 shards, would that present a bottleneck? Only the leader can process multi-shard transactions, but many transactions are multishard? If most contracts operate on their own private data, I can see this being only a theoretical worry- not so if most contracts picked data randomly from the shards.

![](https://ethresear.ch/user_avatar/ethresear.ch/musalbas/45/3251_2.png)

Some questions regarding using merge block fork choice rules to achieve atomic commitment of cross-shard transactions:

*   How can we do synchronous calls between two shards that do not have a direct parent and child relationship? Do we propagate transactions up the tree, requiring shards to process transactions irrelevant to them (thus somewhat defeating the scaling gains achieved by sharding)? Or are you assuming a model for using merge block fork choice rules other than the one linked in the post?
    
*   I imagine that to achieve synchronous calls between the two shards, the parent shard must have an honest majority to actually guarantee that the second half of the transaction will be completed and followed through by the parent shard, once it is executed on the child shard?
    

It would be good to have more discussion and material around merge block fork choice rules and the parent-child shard super-quadratic model of sharding, in general, for more context.

![](https://ethresear.ch/user_avatar/ethresear.ch/vbuterin/45/91_2.png)

> Do we propagate transactions up the tree, requiring shards to process transactions irrelevant to them (thus somewhat defeating the scaling gains achieved by sharding)?

The latter. Though the gains from scaling will still remain. Suppose that we use a scheme which assigns each shard to a pair shard, cycling through all possible pairs (eg. there are `p` shards with prime `p`, in block N we match shards (N mod p, 2N mod p), (3N mod p, 4N mod p), etc). During each block, executors will have to download a witness for every transaction in the corresponding shard, so it increases the amount of work required by at most a factor of 2 (if every transaction is cross-shard), plus a bit more for witness overhead. If we cycle through leader shards that can affect anything, then executors of every other shard will have to execute the transactions of the leader shard first, and executors of the leader shard will have to download witnesses for the portions of the activity that touch other shards, once again increasing the workload at most by a factor of 2 plus witness overhead.