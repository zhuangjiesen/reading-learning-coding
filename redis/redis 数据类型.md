# redis 数据类型

### 官网：
https://redis.io

首页第一段就介绍了 redis 的数据类型

redis 
#### A. string 
#### B. list
#### C. hash
#### D. set
#### F. zset
#### G. bitmap 二进制数组
```
    1.实时分析各种数据
    2.高效高性能高数据关联存储
可以用来做实时访问存储，内存更小
setbit getbit bitcount
```
#### H. HyperLogLog （HLL ）类型
pfadd
pfcount 
计算数据基数
计算唯一字符串的个数
例子：
连续执行
PFADD runoobkey "redis"
PFADD runoobkey "redis"
PFADD runoobkey "redis"
PFADD runoobkey "redis"

pfcount runnnbkey 
结果也是 1


7种redis对象

另外一种是记录左边位置的经纬度和name键
只有命令使用，并没有专门描述的数据结构
geohash的结构实现

GEORADIUS or GEORADIUSBYMEMBER 

GEOADD
GEODIST
GEORADIUS


