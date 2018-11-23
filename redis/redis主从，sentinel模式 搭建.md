# redis主从，sentinel模式 搭建
### redis 目录：

```

/Users/zhuangjiesen/develop/redis/redis-3.2.3/src

cd /Users/zhuangjiesen/develop/redis/redis-3.2.3/src

```


### master 服务器

```
/Users/zhuangjiesen/develop/redis/redis-3.2.3/src/redis-server /Users/zhuangjiesen/develop/redis/config/redis-master.conf
```

### slave 服务

```
redis-server /Users/zhuangjiesen/develop/redis/config/redis-slave-6380.conf

redis-server /Users/zhuangjiesen/develop/redis/config/redis-slave-6381.conf
redis-server /Users/zhuangjiesen/develop/redis/config/redis-slave-6382.conf
redis-server /Users/zhuangjiesen/develop/redis/config/redis-slave-6383.conf


# sentinel 
redis-sentinel /Users/zhuangjiesen/develop/redis/config/sentinel/sentinel-6479.conf --sentinel
redis-sentinel /Users/zhuangjiesen/develop/redis/config/sentinel/sentinel-6480.conf --sentinel
redis-sentinel /Users/zhuangjiesen/develop/redis/config/sentinel/sentinel-6481.conf --sentinel



# 连接redis  
# master服务
redis-cli -h 127.0.0.1 -p 6379 -a redis

# slave 服务
redis-cli -h 127.0.0.1 -p 6380 -a redis
redis-cli -h 127.0.0.1 -p 6381 -a redis
redis-cli -h 127.0.0.1 -p 6382 -a redis
redis-cli -h 127.0.0.1 -p 6383 -a redis
```


### 配置1
```

min-slaves-to-write 2
min-slaves-max-lag 10
当配置了这两个值时，我们可以停掉其中3个slave服务


再对 master 进行操作
```

```

>set age  222
返回
(error) NOREPLICAS Not enough good slaves to write.
即无法进行写操作，因为 slave 失去连接且数量在线的 slave 少于 2


>get age
"222"
读操作可以进行

```

### 当重新启动 slave 时，在进行写操作，操作成功


min-slaves-to-write 2
min-slaves-max-lag 10
这两个参数在 sentinel 模式也有作用，

有三个主机，每个主机分别运行一个redis和一个sentinel:

```

             +-------------+
             | Sentinel 1  | <--- Client A
             | Redis 1 (M) |
             +-------------+
                     |
                     |
+-------------+     |                     +------------+
| Sentinel 2  |-----+-- / partition / ----| Sentinel 3 | <--- Client B
| Redis 2 (S) |                           | Redis 3 (M)|
+-------------+                           +------------+

```


#### 在这个系统中，初始状态下redis3是master, redis1和redis2是slave。之后redis3所在的主机网络不可用了，sentinel1和sentinel2启动了failover并把redis1选举为master。

#### Sentinel集群的特性保证了sentinel1和sentinel2得到了关于master的最新配置。但是sentinel3依然持着的是就的配置，因为它与外界隔离了。

#### 当网络恢复以后，我们知道sentinel3将会更新它的配置。但是，如果客户端所连接的master被网络隔离，会发生什么呢？

#### 客户端将依然可以向redis3写数据，但是当网络恢复后，redis3就会变成redis的一个slave，那么，在网络隔离期间，客户端向redis3写的数据将会丢失。

### 也许你不会希望这个场景发生：

    如果你把redis当做缓存来使用，那么你也许能容忍这部分数据的丢失。

    但如果你把redis当做一个存储系统来使用，你也许就无法容忍这部分数据的丢失了。

#### 因为redis采用的是异步复制，在这样的场景下，没有办法避免数据的丢失。然而，你可以通过以下配置来配置redis3和redis1，使得数据不会丢失。

```
min-slaves-to-write 1
min-slaves-max-lag 10

```

#### 通过上面的配置，当一个redis是master时，如果它不能向至少一个slave写数据(上面的min-slaves-to-write指定了slave的数量)，它将会拒绝接受客户端的写请求。由于复制是异步的，master无法向slave写数据意味着slave要么断开连接了，要么不在指定时间内向master发送同步数据的请求了(上面的min-slaves-max-lag指定了这个时间)。





### sentinel 模式
#### 首先配置 sentinel.xml

#### 发布与订阅信息
#### 客户端可以将 Sentinel 看作是一个只提供了订阅功能的 Redis 服务器： 你不可以使用 PUBLISH 命令向这个服务器发送信息， 但你可以用 SUBSCRIBE 命令或者 PSUBSCRIBE 命令， 通过订阅给定的频道来获取相应的事件提醒。
#### 一个频道能够接收和这个频道的名字相同的事件。 比如说， 名为 +sdown 的频道就可以接收所有实例进入主观下线（SDOWN）状态的事件。
####    通过执行 PSUBSCRIBE * 命令可以接收所有事件信息。
   
   
#### 主要配置如下：

```
 
sentinel monitor mymaster 127.0.0.1 6379 2
# 指示 Sentinel 去监视一个名为 mymaster 的主服务器， 这个主服务器的 IP 地址为 127.0.0.1 ， 端口号为 6379 ， 而将这个主服务器判断为失效至少需要 2 个 Sentinel 同意 （只要同意 Sentinel 的数量不达标，自动故障迁移就不会执行）。
# 不过需要注意的是，无论你设置要多少个 Sentinel 同意才能判断一个服务器失效，一个 Sentinel 都需要获得系统中多数（majority） Sentinel 的支持，才能发起一次自动故障迁移，并预留一个给定的配置纪元 （Configuration Epoch ，一个配置纪元就是一个新主服务器配置的版本号）。也就是说，如果只有少数（minority）Sentinel 进程正常运作的情况下，是不能执行自动故障迁移的。

sentinel auth-pass mymaster redis
# 监听的主服务器的名称以及密码

sentinel down-after-milliseconds mymaster 300
#down-after-milliseconds 选项指定了 Sentinel 认为服务器已经断线所需的毫秒数（判定为主观下线SDOWN）

sentinel failover-timeout mymaster 180000



sentinel notification-script <master-name> <script-path>

```


sentinel 通过向 master 发送 info Replication 命令获取所有slave 服务的信息

sentinel.xml 配置


```
port 6479

dir "/private/tmp"

sentinel monitor mymaster 127.0.0.1 6380 2
sentinel down-after-milliseconds mymaster 300
sentinel failover-timeout mymaster 3000
sentinel auth-pass mymaster redis


```


##### 遇到的坑：
##### 1.主从服务器随时可能因为故障而升级成主服务，主服务重启后会被授权为从服务
##### 在配置文件中需要配置 masterauth "redis" 以免报错，导致服务不可用





