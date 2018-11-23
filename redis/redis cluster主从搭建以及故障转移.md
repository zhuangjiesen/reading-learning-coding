# redis cluster主从搭建以及故障转移

### 在 redis cluster 搭建成功后，高可用的条件下，需要对节点添加从节点

#### 上述的集群环境搭建起来并且槽指派成功，集群运行成功

#### 给 6580 服务设置从节点

```
redis 目录：
/Users/zhuangjiesen/develop/redis/redis-3.2.3/src

cd /Users/zhuangjiesen/develop/redis/redis-3.2.3/src

```

### cluster 从节点启动

```

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-slave-6679.conf

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-slave-6680.conf
	
redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-slave-6681.conf

redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-slave-6682.conf

```




### 启动客户端

```

记得加入 -c 打开集群模式 
就不会出现moved错误不重定向的 槽位负责的服务器上处理
redis-cli -c -h 127.0.0.1 -p 6679 -a redis
redis-cli -c -h 127.0.0.1 -p 6680 -a redis
redis-cli -c -h 127.0.0.1 -p 6681 -a redis
redis-cli -c -h 127.0.0.1 -p 6682 -a redis


```

### 设置为主服务的从节点


```
进入当前从节点客户端
cluster replicate <主节点的id>
成功设置成该主节点的从节点

将 6679 、 6680 、6681 服务设置为 6580 节点的从节点

```


```
cluster keyslot <key>
查看key 会落到哪个槽位

测试 往 6580 槽位添加数据，从节点客户端能获取相应的数据

```


### 故障转移
#### 将 6580 服务退出，从服务会进行选举，得出主节点替换故障主节点，完成故障转移

当从节点发现自己正在复制的主节点进入了已下线状态时，从节点开始对下线主节点进行故障转移
操作步骤：
```
1.复制下线主节点的所有从节点里面，会有一个从节点被选中；
2.被选中的从节点会执行 SLAVEOF no one 命令，成为新的主节点
3.新的主节点会撤销所有对已经下线的主节点的槽指派，并将槽指派指派给自己
4.新住节点会向集群广播 pong 消息，这条Pong 消息可以让集群中的其他节点知道这个节点已经由从节点变成主节点；
5.新的主节点开始接受和自己负责处理的槽有关的命令请求，故障转移完成；

当故障修复成功的主节点被修复后重新上线，会被设置成从节点；


```


