# redis cluster 集群搭建

```
redis 目录：
/Users/zhuangjiesen/develop/redis/redis-3.2.3/src

cd /Users/zhuangjiesen/develop/redis/redis-3.2.3/src

```

### cluster 节点启动

```

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-6579.conf

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-6580.conf
	
redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-6581.conf

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-6582.conf

	

```

##### 建议配置中配置 daemonize no 前台运行，以便于进行故障模拟，并且进行故障转移测试


### 启动客户端

```

记得加入 -c 打开集群模式 
就不会出现moved错误不重定向的 槽位负责的服务器上处理
redis-cli -c -h 127.0.0.1 -p 6579 -a redis
redis-cli -c -h 127.0.0.1 -p 6580 -a redis
redis-cli -c -h 127.0.0.1 -p 6581 -a redis
redis-cli -c -h 127.0.0.1 -p 6582 -a redis


```


### 集群配置
```
在 6579 客户端中
cluster nodes 可以看到把 6579加入集群中

将其他的 6580 、 6581 加入集群中
cluster meet 127.0.0.1 6580
cluster meet 127.0.0.1 6581


cluster nodes 可以打出集群所有节点
cluster forget cluster_Id 可以把节点移除




```


### 槽位指派

```

《redis设计与实现》书上的命令
cluster addslots 0 1 2 3 4 ... 5000
这个命令在redis3.2.3中报错
(error) ERR Invalid or out of range slot

给出另外方法控制台中输入脚本；

for i in {0..5000}

do

	redis-cli -c -p 6579 -a redis cluster addslots $i 

done

可以成功添加槽位

6580 cluster

for i in {5001..10000}

do

	redis-cli -c -p 6580 -a redis cluster addslots $i 

done

6581 cluster

for i in {10001..16383}

do

	redis-cli -c -p 6581 -a redis cluster addslots $i 

done


```


### 只有当 16384 个槽位全部分配给节点后，集群才进入在线状态

### 槽指派成功后进入客户端
```
输入 cluster info 命令查看
>cluster info
cluster_state:ok
//槽位指派完成 0~16383 共16384个槽位
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:2
cluster_my_epoch:2
cluster_stats_messages_sent:37
cluster_stats_messages_received:24


```
```
当输入在某个客户端输入 set age 12 命令时
返回 moved 错误
(error) MOVED 741 127.0.0.1:6579
返回当前key(age) 是属于 741 槽位管理， 属于 127.0.0.1:6579 服务端处理的

注意：
启动客户端时需要加 -c 进入集群模式， 否则redis 只会报错，客户端就不处理重定向操作；
redis-cli -c -h 127.0.0.1 -p 6582 -a redis
这样设置好了之后

在别的节点输入
set age 12
-> Redirected to slot [741] located at 127.0.0.1:6579
结果是重定向并设置成功

```

```
cluster keyslot <key>
查看key 会落到哪个槽位
```

##### 以上就是redis 集群搭建





