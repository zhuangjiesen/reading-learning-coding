# Redis的过期策略(官网文档)

#### How Redis expires keys（Redis怎么处理过期键）：
[过期键官网解释](https://redis.io/commands/expire#how-redis-expires-keys)


#### How Redis expires keys

#### Redis怎么处理过期键？
```
Redis keys are expired in two ways: a passive way, and an active way.
A key is passively expired simply when some client tries to access it, and the key is found to be timed out.
Of course this is not enough as there are expired keys that will never be accessed again. These keys should be expired anyway, so periodically Redis tests a few keys at random among keys with an expire set. All the keys that are already expired are deleted from the keyspace.
Specifically this is what Redis does 10 times per second:
Test 20 random keys from the set of keys with an associated expire.
Delete all the keys found expired.
If more than 25% of keys were expired, start again from step 1.
This is a trivial probabilistic algorithm, basically the assumption is that our sample is representative of the whole key space, and we continue to expire until the percentage of keys that are likely to be expired is under 25%
This means that at any given moment the maximum amount of keys already expired that are using memory is at max equal to max amount of write operations per second divided by 4.

翻译：
Redis有2种方法处理过期键：
1.被动(惰性)过期策略
2.主动过期策略
当客户端访问一个key时，会判断它是否过期；
当然，当没有客户端访问时，这个方法是明显不足的。所以，Redis会阶段性(定时器)随机获取一些过期键进行删除操作；
特别地是，定时清除操作每1秒触发10次
1.检测20个随机的被设置过期属性的键,是否达到过期；
2.将过期键删除；
3.如果超过25%的键都过期，重新执行步骤1；
该算法基于我们取样是在整个键空间中取典型的样品；Redis会持续清理过期键，直到过期键数量小于25%；
这就意味着：每分钟最大数量的过期键处理所消耗的内存最多就等于每4分之一秒最大的写命令的执行。

```


#### How expires are handled in the replication link and AOF file

#### 在主从(master/slave)模式和aof 模式下，怎么处理过期键
```

In order to obtain a correct behavior without sacrificing consistency, when a key expires, a DEL operation is synthesized in both the AOF file and gains all the attached replicas nodes. 
This way the expiration process is centralized in the master instance, and there is no chance of consistency errors.
However while the replicas connected to a master will not expire keys independently (but will wait for the DEL coming from the master),
they'll still take the full state of the expires existing in the dataset, so when a replica is elected to master it will be able to expire the keys independently, fully acting as a master.

翻译：
为了实现一致性，当一个键过期时，一个DEL命令会同步到aof 文件中，并发给从(slave)节点。
这个方法把过期过程集中在了master节点上，没有一致性错误的可能性。
然而所有与master连接的slave节点会市区自身的键失效功能（但是会等待master的DEL命令），
它们(slaves)仍然会管理键的过期事件，所以当从节点(slave)被选中成为master时，它会重新独立地处理过期键。
```

#### [How Redis replication deals with expires on keys](https://redis.io/topics/replication#how-redis-replication-deals-with-expires-on-keys)

```
Redis expires allow keys to have a limited time to live. Such a feature depends on the ability of an instance to count the time, however Redis slaves correctly replicate keys with expires, even when such keys are altered using Lua scripts.
To implement such a feature Redis cannot rely on the ability of the master and slave to have synchronized clocks, since this is a problem that cannot be solved and would result into race conditions and diverging data sets, so Redis uses three main techniques in order to make the replication of expired keys able to work:
Slaves don't expire keys, instead they wait for masters to expire the keys. When a master expires a key (or evict it because of LRU), it synthesizes a DEL command which is transmitted to all the slaves.
However because of master-driven expire, sometimes slaves may still have in memory keys that are already logically expired, since the master was not able to provide the DEL command in time. In order to deal with that the slave uses its logical clock in order to report that a key does not exist only for read operations that don't violate the consistency of the data set (as new commands from the master will arrive). In this way slaves avoid to report logically expired keys are still existing. In practical terms, an HTML fragments cache that uses slaves to scale will avoid returning items that are already older than the desired time to live.
During Lua scripts executions no keys expires are performed. As a Lua script runs, conceptually the time in the master is frozen, so that a given key will either exist or not for all the time the script runs. This prevents keys to expire in the middle of a script, and is needed in order to send the same script to the slave in a way that is guaranteed to have the same effects in the data set.
Once a slave is promoted to a master it will start to expire keys independently, and will not require any help from its old master.

翻译：
Redis过期机制允许键有限制时间内可以访问，依赖于实例可以计算时间的特性，
然而slave节点通过同步键的状态来实现过期，甚至是被lua脚本操作的键；
为了实现Redis无法依赖于master和slave之间的同步时间的问题，
这个问题无法被解决而且会触发竞争状况和数据分叉(出错)，所以Redis用了3个主要的技术为了实现主从状态下过期键的处理工作：
1.从节点们(slaves)并不会过期具体的键，而是等待master把过期键(或者是淘汰策略如LRU )的DEL命令发送过来，然后执行
2.然而，因为主要是依赖于master的过期，有时候在内存中，一些键已经逻辑上过期了(因为master没有及时提供DEL命令)。为了解决这个问题，slave自己使用了它的逻辑时钟，使一个键在读(read operations )操作时，返回空(不存在)，这样并不会破坏数据一致性(因为master一会儿就会把命令传播过来)。这样避免了读取slave时，读取到过期的键。
实际上，有些网络页面的缓存
3.在LUA脚本执行时，过期键策略是无法进行的，在LUA脚本执行期间，master是阻塞的以至于一个键或者存在或者在脚本执行完不存在。这个操作（lua执行）在执行过程中阻止了键的过期，并且这也需要master发送同样的脚本给slave保证数据可靠性。

如果当一个slave升级成为master，它就会自己管理键，并不会依赖于它之前的复制的master

```


