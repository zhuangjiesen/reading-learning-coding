### redis 测试脚本


cpu测试
redis-cli 用法

### 测试网络延迟
三个选项
--latency --latency-history --latency-dist

--latency
./redis-cli -h 172.16.236.163 -p 6379 --latency -i (每几秒一个阶段输出)
用于检测当前客户端到目标redis 的网络延迟


--latency-history
./redis-cli -h 172.16.236.163 -p 6379 --latency-history
每十五秒输出一次网络延迟检测情况

--latency-dist 
./redis-cli -h 172.16.236.163 -p 6379 --latency-dist 
以统计图表的形式输出redis 网络延迟检测情况


检测大内存的key 和 value 通过分段执行scan 方式
--bigkeys 
./redis-cli -h 172.16.236.163 -p 6379 --bigkeys
结果：

```


# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far 'key:000000000019' with 5120 bytes
[00.00%] Biggest hash   found so far 'myset:000000000004' with 1 fields
[24.44%] Biggest hash   found so far 'myset:000000000019' with 4 fields
[24.44%] Biggest list   found so far 'mylist' with 20 items

-------- summary -------

Sampled 45 keys in the keyspace!
Total key length in bytes is 770 (avg len 17.11)

Biggest string found 'key:000000000019' has 5120 bytes
Biggest   list found 'mylist' has 20 items
Biggest   hash found 'myset:000000000019' has 4 fields

32 strings with 102412 bytes (71.11% of keys, avg size 3200.38)
1 lists with 20 items (02.22% of keys, avg size 20.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
12 hashs with 21 fields (26.67% of keys, avg size 1.75)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```


### redis-server 
内存容量测试,检测分配的内存容量是否稳定
./redis-server -h 172.16.236.163 -p 6379 --test-memory 1024

### redis-benchmark 用法
基准测试
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 200000
100个客户端同时请求redis 一共执行200000次， 会对各种数据结构的命令进行测试

./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 20 -r 20 -d 1024

用的虚拟机
内存 4g
cpu 1核2.7 GHz Intel Core i5
系统 centos 7

```
设置随机的string 的 key 的value 为 20480 字节
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 20 -r 20 -d 20480

100个客户端 200000个请求测试 get 
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 200000 -r 20 -t get

结果:
====== GET ======
  200000 requests completed in 59.48 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1
0.00% <= 4 milliseconds
0.00% <= 5 milliseconds
....
10.01% <= 24 milliseconds
...
99.99% <= 120 milliseconds
100.00% <= 121 milliseconds
100.00% <= 122 milliseconds
100.00% <= 122 milliseconds
3362.64 requests per second
```

```
设置随机的string 的 key 的value 为 1024 字节
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 20 -r 20 -d 1024
100个客户端 200000个请求测试 get
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 200000 -r 20 -t get
结果：
====== GET ======
  200000 requests completed in 14.11 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1

0.00% <= 4 milliseconds
0.02% <= 5 milliseconds
57.26% <= 6 milliseconds
...
99.99% <= 37 milliseconds
100.00% <= 38 milliseconds
100.00% <= 39 milliseconds
14176.35 requests per second

```


```
设置随机的string 的 key 的value 为 5120 字节
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 20 -r 20 -d 5120
100个客户端 200000 个请求测试 get
./redis-benchmark -h 172.16.236.163 -p 6379 -c 100 -n 200000 -r 20 -t get
结果：
====== GET ======
  200000 requests completed in 15.91 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1

0.00% <= 3 milliseconds
0.00% <= 4 milliseconds
0.02% <= 5 milliseconds
...
99.95% <= 21 milliseconds
100.00% <= 22 milliseconds
12569.13 requests per second
```

value 越大性能越差，所以避免大key 和 大 value 导致阻塞
而且并发请求到达这个情况下，通过 top 命令看到 redis 进程消耗cpu 接近百分百

```
请求数到达 2000000 时，cpu 已经接近100
但是 load average 指标并不高
./redis-benchmark -h 172.16.236.163 -p 6379 -c 200 -n 1000000 -r 20 -t get
```


