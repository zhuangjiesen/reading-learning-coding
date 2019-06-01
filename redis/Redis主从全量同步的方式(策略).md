# Redis主从全量同步的方式(策略)
[github博客地址](https://github.com/zhuangjiesen/reading-learning-coding/blob/master/redis/Redis主从全量同步的方式(策略).md)

### redis.conf配置中发现的
#### 注: 版本 2.8.9 3.2.0 4.0.0 都有的配置
### 本来以为就是生成rdb文件，传输给slave，其实不然

```
# Replication SYNC strategy: disk or socket.
#

# WARNING: DISKLESS REPLICATION IS EXPERIMENTAL CURRENTLY
# -------------------------------------------------------
#
# New slaves and reconnecting slaves that are not able to continue the replication
# process just receiving differences, need to do what is called a "full
# synchronization". An RDB file is transmitted from the master to the slaves.
# The transmission can happen in two different ways:
#
# 1) Disk-backed: The Redis master creates a new process that writes the RDB
#                 file on disk. Later the file is transferred by the parent
#                 process to the slaves incrementally.
# 2) Diskless: The Redis master creates a new process that directly writes the
#              RDB file to slave sockets, without touching the disk at all.
#
# With disk-backed replication, while the RDB file is generated, more slaves
# can be queued and served with the RDB file as soon as the current child producing
# the RDB file finishes its work. With diskless replication instead once
# the transfer starts, new slaves arriving will be queued and a new transfer
# will start when the current one terminates.
#
# When diskless replication is used, the master waits a configurable amount of
# time (in seconds) before starting the transfer in the hope that multiple slaves
# will arrive and the transfer can be parallelized.
#
# With slow disks and fast (large bandwidth) networks, diskless replication
# works better.
```

### 原文翻译：
```

redis主从复制的方式：
主从同步策略： 硬盘 或者 socket(直接网络 DISKLESS)

警告！：DISKLESS 同步方式当前处理实验阶段

新的从节点和断线重连上，无法继续同步数据的从节点(数据不一致),需要出发全同步步骤(full
# synchronization)。 master还是通过rdb 传输给slave，但是方式(实现) 可以分成2种方式：
1. Disk-backed:
	redis-master 创建新的进程，触发rdb生成rdb文件持久化到硬盘，父进程将文件传输给slaves

2. Diskless:
	redis-master 创建新的进程, 触发rdb，但是不生成rdb文件(但是一样获取rdb快照数据)，写到slave的socket（直接传输给slave）,并不会生成持久化的rdb文件

使用 disk-backed 同步，当rdb文件生成时，子进程一完成rdb文件生成时所有排队中的slaves都能收到rdb 文件。而 diskless ，每次只会给一个slave同步数据,当同步结束后后续到达的slave，可能需要重新同步

使用 diskless 同步，master会等待一个可以配置的时间(几秒内)，为了多个slave一起进行 diskless 同步

硬盘性能差，网络性能好的情况下 diskless 效果更佳
```


