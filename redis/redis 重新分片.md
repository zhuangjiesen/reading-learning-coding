# redis 重新分片

### 这种场景肯定能遇到：
比如分配了3个节点，接着新增节点，需要进行重新分片，即重新分配槽位

### 新增节点 6582 
```
redis 目录：
/Users/zhuangjiesen/develop/redis/redis-3.2.3/src

cd /Users/zhuangjiesen/develop/redis/redis-3.2.3/src

启动服务器
redis-server /Users/zhuangjiesen/develop/redis/config/cluster/redis-cluster-slave-6682.conf


在主节点 6579 中加入该节点，进入集群
cluster meet 127.0.0.1 6582




```
#### 接着就看到进入集群了，但是该节点没有分配槽点，这时候节点可以有两种用法：
1. 通过作为某个节点的从节点，进行故障检测并进行故障转移
2. 作为集群另一个节点，需要重新分配槽位；


### 接下来就讲迁移槽和数据：


#### 流程如下：
##### 1. 在目标节点客户端中执行命令：


```
cluster setslot {slot} importing {sourceNodeId} 
```
##### 让目标节点准备导入槽数据的数据，

##### 2. 对源节点(被迁出数据的服务器) 的客户端中执行命令 ：


```
cluster setslot migrating {targetNodeId} 
```
##### 让源节点准备迁出槽数据


##### 3. 源节点中执行命令
```
cluster getkeysinslot {slot} {count}
```
##### 让源节点输出该槽位的 count数目的 key 

##### 4.在源节点上执行命令：
```
 migrate {targetIp} {targetport} "" 0 {timeout} keys {keys...}
 
 例子1：
  migrate 127.0.0.1 6582 "" 0 5000 keys replacekey
  例子2：
    migrate 127.0.0.1 6582 "" 0 5000 keys key1 key2 key3
```

##### 将槽上的key 中的数据迁移到目标节点上，否则槽转移时会报错

```
执行：
cluster setslot 4423 node 510950fef54eddf39e9ab71a33efb03b2154907b
报错：
(error) ERR Can't assign hashslot 4423 to a different node while I still hold keys for this hash slot.
```
##### 你还持有这些数据，无法进行转移

##### 5.重复或批量迁移直到所有都迁移到新节点（目标节点）
##### 最后，想 所有（每个节点） 主节点发送(执行)命令
```
cluster setslot {slot} node {targetNodeId} 
例子：
cluster setslot 4423 node 510950fef54eddf39e9ab71a33efb03b2154907b
```

##### 保证槽节点映射变更及时传播，需要遍历发送给所有主节点，更新被迁移的槽指向，否则新节点和旧节点都是不可用的
##### 就是说 6581 还不知道 4423 这个槽位已经由 6582 服务进行管理，还只知道是由原来的 6579 管理，这样子插入或者操作会报错




#### 槽位上没有数据当然可以直接通知各个主节点变更事件。


### 测试小帮助
```
用 
cluster keyslot zhuangjiesen(或者你要测试的key)

127.0.0.1:6580> cluster keyslot replacekey
(integer) 10053
命令
查看槽点位置 10053 ，然后可以通过 迁移 10053 槽位，进行重新分片的测试


```

#### 最后分片完
```
127.0.0.1:6579> cluster nodes
0b8bd407726a0737770458d8aadb3f3176a41cc3 127.0.0.1:6581 master - 0 1494761151496 2 connected 10001-10052 10054-16383
3d5f6827a0e2e0c6cd146213797c2e9897a01b95 127.0.0.1:6579 myself,master - 0 0 1 connected 0-5000 [4423->-510950fef54eddf39e9ab71a33efb03b2154907b]
b7bac804f3654f3bdbe38c7747d7db4b05f84cd4 127.0.0.1:6580 master - 0 1494761152526 0 connected 5001-5904 5906-10000
510950fef54eddf39e9ab71a33efb03b2154907b 127.0.0.1:6582 master - 0 1494761153560 3 connected 5905 10053

6582 服务获得 5905 10053 
6581 服务则取消了这两个槽位的管理
6582 客户端获取 这两个槽位数据时也不会返回 moved 错误
```


