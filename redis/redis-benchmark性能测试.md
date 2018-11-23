

4m = 1024*1024*4 bit = 4194304 bit
1g = 1024*1024*1024 bit = 
随机插入数据
/home/redis/redis-3.2.9/src/redis-benchmark -h 192.168.130.130 -p 6379 -a redis -t set -n 50 -d 20971520 -c 10 -r 100

/home/redis/redis-3.2.9/src/redis-benchmark -h 192.168.130.130 -p 6379 -a redis -t set -n 50 -d 4194304 -c 10 -r 100

随机get 
/home/redis/redis-3.2.9/src/redis-benchmark -h 192.168.130.130 -p 6379 -a redis -t get -c 50 -n 1000 -r 100

```

以下参数被支持：

Usage: redis-benchmark [-h <host>] [-p <port>] [-c <clients>] [-n <requests]> [-k <boolean>]

 -h <hostname>      Server hostname (default 127.0.0.1)
 -p <port>          Server port (default 6379)
 -s <socket>        Server socket (overrides host and port)
 -a <password>      Password for Redis Auth
 -c <clients>       Number of parallel connections (default 50)
 -n <requests>      Total number of requests (default 100000)
 -d <size>          Data size of SET/GET value in bytes (default 2)
 -dbnum <db>        SELECT the specified db number (default 0)
 -k <boolean>       1=keep alive 0=reconnect (default 1)
 -r <keyspacelen>   Use random keys for SET/GET/INCR, random values for SADD
  Using this option the benchmark will expand the string __rand_int__
  inside an argument with a 12 digits number in the specified range
  from 0 to keyspacelen-1. The substitution changes every time a command
  is executed. Default tests use this to hit random keys in the
  specified range.
 -P <numreq>        Pipeline <numreq> requests. Default 1 (no pipeline).
 -q                 Quiet. Just show query/sec values
 --csv              Output in CSV format
 -l                 Loop. Run the tests forever
 -t <tests>         Only run the comma separated list of tests. The test
                    names are the same as the ones produced as output.
 -I                 Idle mode. Just open N idle connections and wait.


```



slowlog 命令

配置：
 slowlog-log-slower-than 100
 操作大于 100 微妙记录到慢查询日志
 slowlog-max-len 1000
 慢查询日志最多保存 1000条日志


格式：
 ```
redis> SLOWLOG GET
1) 1) (integer) 12                      # 唯一性(unique)的日志标识符 (id )
   2) (integer) 1324097834              # 被记录命令的执行时间点，以 UNIX 时间戳格式表示
   3) (integer) 16                      # 查询执行时间，以微秒为单位
   4) 1) "CONFIG"                       # 执行的命令，以数组的形式排列
      2) "GET"                          # 这里完整的命令是 CONFIG GET slowlog-log-slower-than
      3) "slowlog-log-slower-than"

 ```

 命令：
 慢查询日志长度：
  SLOWLOG LEN

慢查询日志清空：
  SLOWLOG RESET
获取慢查询列表
SLOWLOG GET <数量>