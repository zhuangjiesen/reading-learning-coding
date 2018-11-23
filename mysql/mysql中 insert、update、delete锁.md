# mysql中 insert、update、delete锁

### MySQL 版本： 5.7.17-log 
### 对于表的锁的探索
#### 开多个客户端界面
```
DROP TABLE IF EXISTS `m_user`;
CREATE TABLE `m_user` (
  `i_id` int(10) unsigned zerofill NOT NULL AUTO_INCREMENT,
  `i_name` varchar(255) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  `is_delete` varchar(1) DEFAULT NULL,
  `i_type` varchar(5) DEFAULT NULL,
  PRIMARY KEY (`i_id`),
  UNIQUE KEY `index_i_id` (`i_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

```
#### 测试数据:
```

INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000001', 'ajason', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '1');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000002', 'tom', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '2');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000003', 'jane', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '3');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000004', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '4');

```
#####  i_type 设置为 unique 的 btree 索引时
```
index_i_type	i_type	Unique
```

### 若字段条件(where 条件)的值在表中并无记录，是不会触发锁的 (即影响行数0)

### 事务隔离级别：

```
select @@tx_isolation

结果：
REPEATABLE-READ
```


### alter table 操作，不需要使用事务 (也没有作用) , 遇到表中未提交的事务会被阻塞



#### 事务开启：begin ;
#### 事务回归：rollback ;
#### 事务提交：commit ;

### update 锁
```
事务A 中执行update 时( 未提交commit )，若使用主键查询，锁的是行记录
1. update m_user set i_name = 'ajason' where i_id = 1;
2. update m_user set i_name = 'ajason' where i_id in (1  , 2);

事务B 中
使用 insert delete update 对 1.语句中 i_id = 1 行进行操作时，会发生阻塞
使用 insert delete update 对 2.语句中 i_id = 1 与 2 的行进行操作时，会发生阻塞

============间隙锁1==========
事务A：
3. update m_user set i_name = 'ajason' where i_id in (1  , 7 );
事务B
3. update m_user set i_name = 'ajason' where i_id = 3;
事务B 并不会阻塞
所以通过主键进行的update 都是单行的


```

### 非主键索引问题 
#### 若用唯一索引作为条件,锁的也是数据行 
如：
```
1.单行update操作
事务A：
update m_user set i_name = 'ajason333' where i_type in ( '2' )
事务B：
update m_user set i_name = 'ajason888' where i_type in ( '3' )
结果： 不阻塞

2.单行update操作
事务A：
update m_user set i_name = 'ajason333' where i_type in ( '2' )
事务B：
update m_user set i_name = 'ajason888' where i_type in ( '2' )
结果： 事务B阻塞

3.多行定位 update操作
事务A：
update m_user set i_name = 'ajason333' where i_type in ( '1' , '4' )
事务B：
update m_user set i_name = 'ajason888' where i_type in ( '2' )
结果： 不阻塞
事务C：
update m_user set i_name = 'ajason888' where i_type in ( '4' )
结果： 事务C阻塞


4. 范围查询的 update 
A1:
	事务A：
	update m_user set i_name = 'ajason333' where i_type like '1%'
	事务B：
	update m_user set i_name = 'ajason888' where i_type in ( '2' )
	结果： 事务B阻塞
A2:
	事务A：
	update m_user set i_name = 'ajason333' where i_type > '5'
	事务B1：
	update m_user set i_name = 'ajason888' where i_type in ( '2' )
	结果： 不阻塞
	事务B2：
	update m_user set i_name = 'ajason888' where i_type in ( '6' )
	结果： 事务B阻塞

```
结论，唯一索引作为查询条件进行Update 操作时，锁定的数据都是行(具体检索到哪个行，就锁定该行)


#### 若用非主键查询(普通列或普通索引)进行条件查询，锁的是整表，这时候事务还未提交时，插入(update)删除(delete)操作都阻塞等待
后果：容易死锁 ， 导致整表操作性能瓶颈
如：
```
1.
事务A：
update m_user set i_name = 'ajason666' where is_delete = '2'
事务B：
update m_user set i_name = 'ajason777' where is_delete = '6'
结果事务B 阻塞

事务A 中执行( 未提交 )：update m_user set i_name = 'ajason' where i_type in ( ‘1’ )
其他事务中对表中数据进行的 insert , update , delete 操作都会被阻塞
```


同理 ：

### delete 锁
#### 事务中执行非唯一索引查询锁表，主键/唯一索引查询锁行
```
delete from  m_user  where i_type in ( '1' )
delete from  m_user where i_id = 1;
```

### 当表中还有事务 未提交
#### alter table 对表进行修改会造成死锁




### insert锁
#### 当事务A执行如下语句时(i_id 为 5 ,原来数据库没有的字段)

```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
```

#### 事务A未提交
##### 1.其他事务对i_id 为 5 的操作无法继续(阻塞等待)
比如执行(同理delete)
```
1.行锁：update m_user set i_name = 'ajason' where i_id = 5;
2.表锁：update m_user set i_name = 'ajason' where i_type = 1  (不论i_type是多少)

------------------==================注意====================----------------------------
这里语句2问题
i_type 类型是varchar 所以语句问题导致表锁

new2.表锁：update m_user set i_name = 'ajason' where i_type = '2'  
则 new2时，如果不是同一索引不会触发事务阻塞


```
##### 2.其他事务对i_id 不为 5 的操作不会阻塞
比如执行(同理delete)

```
行锁：update m_user set i_name = 'ajason' where i_id = 6;
```

##### 或者执行 insert 操作 分3种情况分析：
1.主键与unique索引都相同
```
事务B 执行:
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
两个事务都阻塞，晚提交的事务报错
```

2.主键相同，索引不同

```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '6');
两个事务都阻塞，晚提交的事务报错
```

3.主键不同，索引相同
```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000006', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
两个事务都阻塞，晚提交的事务报错
```

4.主键不同，索引不同
```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000006', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '6');
成功提交

```

4.主键和索引都不同
互不影响



### 结论：
#### 说明行锁通过主键/唯一索引， 普通字段或非唯一索引进行操作只会锁表


### 锁表时：
```
使用 insert , update , delete 语句对表操作时，都会阻塞
锁行：
对该行进行操作时才会阻塞
```

### 锁表和锁行时：
#### 进行普通 select 查询并不会阻塞
### select 锁(排他锁和共享锁)


### for update 
#### 1. 根据普通字段(普通非唯一索引) is_delete 
```
事务A：
select * from m_user  where 1=1 and is_delete = '2' for update
事务B：
B1. update m_user set i_name = 'ajason64444' where is_delete = '6'
B2. update m_user set i_name = 'ajason888' where i_id in ( 2 )
B3. update m_user set i_name = 'ajason888' where i_type in ( '2' )
结果：任意操作事务B 都是阻塞的
结论：对于普通字段进行 X LOCK 操作，是锁表的
```

#### 2. 根据唯一索引字段 i_type 
```
事务A：
select * from m_user  where 1=1 and i_type in ( '4' ) for update
事务B：
B1.update m_user set i_name = 'ajason888' where i_type in ( '2' )
B2.update m_user set i_name = 'ajason888' where i_type in ( '6' )
B3.update m_user set i_name = 'ajason888' where i_type in ( '4' )
结果： B1 B2 不阻塞 ，B3阻塞事务
结论：唯一索引字段X LOCK 操作是锁行的
```

#### 3. 唯一索引，定位多行条件，测试间隙锁
```
3.A.1：
	事务A：
	select * from m_user where 1=1 and i_type in ( '1' , '4' ) for update
	事务B：
	B1.update m_user set i_name = 'ajason888' where i_type in ( '2' )
	B2.update m_user set i_name = 'ajason888' where i_type in ( '6' )
	B3.update m_user set i_name = 'ajason888' where i_type in ( '4' )
	B4.update m_user set i_name = 'ajason888' where i_id in ( 2 )
	B5.update m_user set i_name = 'ajason888' where i_id in ( 6 )
	结果：事务B 所有都阻塞
	结论：对于唯一索引，定位多行记录的 X LOCK 查询 ，会进行锁表操作，其他事务均会阻塞
3.A.2：
	事务A (主要测试间隙锁，这里就定位相邻的两个记录，但是结果依然是整个表无法操作)：
	select * from m_user where 1=1 and i_type in ( '1' , '2' ) for update
	事务B：
	B1.update m_user set i_name = 'ajason888' where i_type in ( '2' )
	B2.update m_user set i_name = 'ajason888' where i_type in ( '6' )
	B3.update m_user set i_name = 'ajason888' where i_type in ( '4' )
	B4.update m_user set i_name = 'ajason888' where i_id in ( 2 )
	B5.update m_user set i_name = 'ajason888' where i_id in ( 6 )
	结果：事务B 所有都阻塞
	结论：对于唯一索引，定位多行记录的 X LOCK 查询 ，会进行锁表操作，其他事务均会阻塞
```

#### 4. 唯一索引的范围查询 

```
事务A：
select * from m_user where 1=1 and i_type > '4' for update
事务B：
update m_user set i_name = 'ajason64444' where is_delete = '6'
update m_user set i_name = 'ajason888' where i_id in ( 2 )
update m_user set i_name = 'ajason888' where i_id in ( 6 )
update m_user set i_name = 'ajason888' where i_id in ( 7 )
update m_user set i_name = 'ajason888' where i_type in ( '2' )
update m_user set i_name = 'ajason888' where i_type in ( '6' )
select * from m_user where 1=1 and i_type = '1' for update
结果：事务B 全部阻塞
结论：对于唯一索引，X LOCK 进行范围查询的话，会对表进行表锁，其他事务都会阻塞
```

#### 5. 主键范围查询
```
事务A：
select * from m_user where 1=1 and i_id > 4 for update
事务B：
B1.update m_user set i_name = 'ajason64444' where is_delete = '6'
B2.update m_user set i_name = 'ajason888' where i_id in ( 2 )
B3.update m_user set i_name = 'ajason888' where i_id in ( 6 )
B4.update m_user set i_name = 'ajason888' where i_id in ( 7 )
B5.update m_user set i_name = 'ajason888' where i_type in ( '2' )
B6.update m_user set i_name = 'ajason888' where i_type in ( '6' )
B7.select * from m_user where 1=1 and i_type = '1' for update
结果：B1 B3 B4 B7 阻塞  B2 B5 B6 不阻塞
结论：主键索引的范围查询， X LOCK条件下，还是锁行的，与唯一索引不一样
```

#### 6. 主键的单行查询
```
事务A：
select * from m_user where 1=1 and i_id = 2 for update
事务B
B1.update m_user set i_name = 'ajason64444' where is_delete = '6'
B2.update m_user set i_name = 'ajason888' where i_id in ( 2 )
B3.update m_user set i_name = 'ajason888' where i_id in ( 6 )
B4.update m_user set i_name = 'ajason888' where i_id in ( 7 )
B5.update m_user set i_name = 'ajason888' where i_type in ( '2' )
B6.update m_user set i_name = 'ajason888' where i_type in ( '6' )
B7.select * from m_user where 1=1 and i_type = '1' for update
结果：B1 B2 B5 阻塞 其他不阻塞 
结论：单行查询的X LOCK 锁行
```

#### 6. 主键的多行查询
```
事务A：
select * from m_user  where 1=1 and i_id in ( 1 , 4 ) for update
事务B
B1.update m_user set i_name = 'ajason64444' where is_delete = '6'
B2.update m_user set i_name = 'ajason888' where i_id in ( 2 )
B3.update m_user set i_name = 'ajason888' where i_id in ( 1 )
B4.update m_user set i_name = 'ajason888' where i_id in ( 4 )
B5.update m_user set i_name = 'ajason888' where i_id in ( 6 )
B6.update m_user set i_name = 'ajason888' where i_id in ( 7 )
B7.update m_user set i_name = 'ajason888' where i_type in ( '2' )
B8.update m_user set i_name = 'ajason888' where i_type in ( '6' )
B9.select * from m_user where 1=1 and i_type = '1' for update
结果：B1 B3 B4 阻塞 其他不阻塞 
结论：单行查询的X LOCK 锁行
```

同理：
```
语句：
select * from m_user  where 1=1 and i_id in ( 1 , 3 , 5 ) for update
测试结果也是锁行
```


### select 锁的结论
```
非：
select * from  m_user (条件) for update
select * from  m_user (条件) lock in share mode
加条件的时候就要区分行锁表锁的select 查询了


1.主键 范围查询，多行，单行查询条件下都是行锁
2.unique索引下 单行查询是行锁 多行查询，范围查询下都是表锁
3.其他条件下，都触发了表锁

```

#### 数据库实现了超时锁释放
#### 数据库锁超时报错：
```
[Err] 1205 - Lock wait timeout exceeded; try restarting transaction
```

### 操作:
#### 当遇到alter table 对表操作时，直接无响应 或者死锁
```
select * from information_schema.INNODB_TRX
select * from information_schema.INNODB_LOCKS
select * from information_schema.INNODB_LOCK_WAITS
show PROCESSLIST

杀掉进程
kill 45(processlist 的 id )
接着提交回滚未完成的事务，然后继续对表操作
```



### 总结

```
update delete 操作时，尽量是通过主键进行检索，如果是条件查询的话可以先通过select 语句查询出结果，再进行update 操作，
防止锁表操作，导致其他写操作发生性能问题

select 操作的话尽量避免用X LOCK 锁，可以使用别的方法实现一样的效果，并不需要用到数据库锁

```