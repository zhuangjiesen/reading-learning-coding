# redis 定时器 serverCron 函数

### redis 服务器中的 serverCron 函数每100毫秒执行一次

#### 1.更新服务器时间
#### 2.更新 lru 时钟
#### 3.更新服务器每秒命令执行次数
#### 4.更新服务器内存峰值

```
执行命令：
info memory 
可以看到内存结果：
used_memory_peak
used_memory_peak_human（这个是直观的表示）

```
#### 5.处理SIGTERM信号，服务器关闭，关闭前会根据配置执行 rdb 持久化操作
#### 管理客户端资源
1.释放超时客户端
2.重新创建输入缓冲区

#### 管理数据库资源 databasesCron 函数，对一部分数据库进行检查，删除其中过期键，并在需要的时候进行字典收缩操作(ziplist ,dictionary)

#### 执行被延迟的 BGREWRITEAOF命令
#### 检查持久化操作的运行状态

#### 将aof 缓冲区的内容写入 aof 文件
#### 关闭异步客户端
#### 增加cronloops 计数器的值，记录serverCron函数执行的次数


