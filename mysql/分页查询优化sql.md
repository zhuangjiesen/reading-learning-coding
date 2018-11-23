# 优化sql 
### 建order by 的索引

创建索引:
```
	CREATE INDEX "index_pass_time" ON "public"."vehicle_record" USING btree (c_pass_time);
	CREATE INDEX "index_plate_info_color" ON "public"."vehicle_record" USING btree (c_plate_info, i_plate_color);
```

#### 测试在 500w + 条数据，进行 insert 或者 update操作 耗时不超过1 秒
#### 如果需要update ，要先select 查询，分页进行updae操作，否则数据库会阻塞

### 缓存count 值
后台将count 结果缓存起来， 继承了linkedHashMap 类，实现lru 算法的count结果缓存，并可以设置缓存失效时间
刷新第一页时不走 count数值缓存，保证能取到新数据；
VeicleRecordCountCacheHelper.java


### 改写sql 


### 优化了 LIMIT end OFFSET start 
原因 当 start值超过50w 后 查询结果线性下降

当执行：
```
EXPLAIN ANALYSE select   vr.*  from  vehicle_record vr   
where 1=1  and  i_cross_status != 2  and  c_pass_time < '2017-12-28 23:22:09'   
order by c_pass_time desc 
limit 10  OFFSET 1000000;
```
数据库直接卡死

优化：
取消跳转页面操作
下一页的查询不用 offset 定位用 上一页最后一条记录的 c_pass_time(索引) 作为属性

例子：
获取第二页的数据：
```
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2   order by c_pass_time desc limit 10 offset 20;
```
也可以改写成:
```
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2  and  c_pass_time < '2017-12-28 22:51:36'   order by c_pass_time desc limit 10;
( c_pass_time < '2017-12-28 22:51:36'  这个值是上一页面查询结果的最后一条)
```

虽然刚开始不会有性能差异的感知

但是，到几万条数据开始 比如要算 limit 10 offset 50000 :

```
这句用来查出 limit 10 offset 49990 最后一条的值，复制到下一行sql作为条件，取消offset
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2   order by c_pass_time desc limit 10 offset 49990;

EXPLAIN ANALYSE
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2  and  c_pass_time < '2017-11-26 06:13:20'   order by c_pass_time desc limit 10;
运行结果 0.009 秒

EXPLAIN ANALYSE
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2   order by c_pass_time desc limit 10 offset 50000;
运行结果 0.662 秒 (而且是扫描了 Index Scan 之后的结果)

```

当offset 继续大下去 ， 因为表中有百万数据 
性能差距越来越明显 当执行到 100w 也就是 10w页时 :

```

（用来获取上一页最后记录用）
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2   order by c_pass_time desc limit 10 offset 999990;

EXPLAIN ANALYSE
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2  and  c_pass_time < '2016-12-12 01:52:10'   order by c_pass_time desc limit 10;
运行结果 0.0018 秒

EXPLAIN ANALYSE
select   vr.*  from  vehicle_record vr   where 1=1  and  i_cross_status != 2   order by c_pass_time desc limit 10 offset 1000000;
运行结果 35 秒 (而且是扫描了 Index Scan 之后的结果)
```


所以优化的方式就是不用 offset （只在第一页使用）
其他进行下一页查询时都将上一页的最后一条记录传至后台，用来作offset用途
同理
返回上一页操作也一个意思






