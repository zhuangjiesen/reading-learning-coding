# redis SLAVE过期键策略
官方文档：https://redis.io/topics/replication#how-redis-replication-deals-with-expires-on-keys
### 原文截取：
```
How Redis replication deals with expires on keys
Redis expires allow keys to have a limited time to live. Such a feature depends on the ability of an instance to count the time, however Redis slaves correctly replicate keys with expires, even when such keys are altered using Lua scripts.
To implement such a feature Redis cannot rely on the ability of the master and slave to have synchronized clocks, since this is a problem that cannot be solved and would result into race conditions and diverging data sets, so Redis uses three main techniques in order to make the replication of expired keys able to work:
1.Slaves don't expire keys, instead they wait for masters to expire the keys. When a master expires a key (or evict it because of LRU), it synthesizes a DEL command which is transmitted to all the slaves.
2.However because of master-driven expire, sometimes slaves may still have in memory keys that are already logically expired, since the master was not able to provide the DEL command in time. In order to deal with that the slave uses its logical clock in order to report that a key does not exist only for read operations that don't violate the consistency of the data set (as new commands from the master will arrive). In this way slaves avoid to report logically expired keys are still existing. In practical terms, an HTML fragments cache that uses slaves to scale will avoid returning items that are already older than the desired time to live.
3.During Lua scripts executions no keys expires are performed. As a Lua script runs, conceptually the time in the master is frozen, so that a given key will either exist or not for all the time the script runs. This prevents keys to expire in the middle of a script, and is needed in order to send the same script to the slave in a way that is guaranteed to have the same effects in the data set.
Once a slave is promoted to a master it will start to expire keys independently, and will not require any help from its old master.
```

### 翻译：
```
Redis同步如何处理键过期
redis可以配置键的过期(限制时间内存活)时间，这样一个功能是依赖于redis实例的时间计算。然而,redis 的slaves 正确地(准确无误地)从master将带有过期属性的键复制过来，甚至当这些键通过lua脚本修改后。
为了实现(redis无法依赖master和slave之间的同步时钟)的功能，因为这是个无法解决的问题且会导致竞争条件(比如lua脚本执行期间无法同步命令给slave )和不同的数据集(数据不一致)，所以redis 用了3个主要的技术来解决主从复制中的键过期：
1.slave不会主动触发键过期，相反slave等待master来同步键过去。当master需要将键过期(或者键由于LRU过期)，它(master)同步了一个DEL命令给所有的slave节点。
2.然后，由于是master驱动的过期事件而且由于master无法及时的提供(传播)DEL命令，有时候slave内存中依然存在实际上(逻辑上)已经过期的键。为了解决这个问题，slave使用独自的逻辑时钟来声明键不存在当且仅当读操作(不破坏数据集的一致性情况下-> 因为master的命令始终会(将会)到达)。slave通过这个方法避免客户端读取到已经不存在的数据。在现实案例中，html页面通过使用slave缓存页面来规模化(扩容)，可以防止页面过期不及时的情况。
3.在lua脚本执行期间(master)因为lua脚本执行是阻塞的，所以所有的键过期事件不会触发。一旦lua脚本执行，master概念上的服务器时间是停滞的，以至于一个key会在Lua执行脚本期间会始终存在。所以master会发送同样的Lua脚本，保证数据同步(保证得到相同效果的数据集)
一旦一个slave被晋升成master，它将能独立地处理键失效，而且不再需要老的master的任何帮助

```

