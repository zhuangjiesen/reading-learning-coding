### 联合索引

表结构
```

DROP TABLE IF EXISTS `goods`;
CREATE TABLE `goods` (
  `goods_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `goods_name` varchar(128) DEFAULT NULL,
  `goods_create_user` varchar(128) DEFAULT NULL,
  `goods_create_time` datetime DEFAULT NULL,
  `goods_qty` int(11) DEFAULT NULL,
  `goods_is_remain` varchar(1) DEFAULT NULL,
  `goods_type` varchar(1) DEFAULT NULL,
  PRIMARY KEY (`goods_id`),
  KEY `index_user_id` (`goods_create_user`) USING BTREE,
  KEY `index_goods_name` (`goods_type`,`goods_name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=300002 DEFAULT CHARSET=utf8;


这时候表中样品数据有30w 条记录， goods_type 字段的值分布是 1~9
goods_type , goods_name 是联合索引，这时候要测试联合索引的执行计划

```


#### 介绍一下查询场景，现在只想通过 goods_name 去查询相应的记录

#### 查询A :

```

explain
select goods_name 
from goods 
where 1=1 
and goods_name = '商品名称_31501550617221'

执行时间 0.032s
执行计划:
id select_type  table  partitions  type   possible_keys  key                 key_len  ref   rows    filtered  Extra  
1   SIMPLE      goods  Null        index  Null           index_goods_name    393      Null  313590  10        Using where;  Using index


结论:
因为只取 goods_name 列数据，所以数据库取了覆盖索引数据

```

#### 查询A1 :

```

explain
select * 
from goods 
where 1=1 
and goods_name = '商品名称_31501550617221'

执行时间 6.361s
执行计划:
id select_type  table  partitions  type   possible_keys  key    key_len  ref   rows    filtered  Extra  
1  SIMPLE       goods  Null        ALL    Null           Null   Null     Null  313590  10        Using where

结论:未使用索引，全表扫描，执行效率低

```






#### 查询B :
```

explain
select goods_name 
from goods 
where 1=1 
and goods_type in ( '1' , '2' ,'3' ,'4' ,'5' ,'6' ,'7' ,'8' ,'9' )
and goods_name = '商品名称_31501550617221'

执行时间 0.001s
执行计划:
id select_type  table partitions type    possible_keys      key               key_len ref   rows filtered  Extra  
1  SIMPLE       goods Null       range   index_goods_name   index_goods_name  393     Null  10   100       Using where; Using index


结论: 使用索引效率高

```
 



#### 查询B1 :
```

explain
select * 
from goods 
where 1=1 
and goods_name = '商品名称_31501550617221'
and goods_type in ( '1' , '2' ,'3' ,'4' ,'5' ,'6' ,'7' ,'8' ,'9' )

执行时间 0.062s
执行计划:
id select_type  table partitions type    possible_keys      key               key_len ref   rows filtered  Extra  
1  SIMPLE       goods Null       range   index_goods_name   index_goods_name  393     Null  10   100       Using index condition

结论: 联合索引的条件，在where 语句中的顺序并没有关系

```




#### 查询B2 (模糊查询)：
```
explain
select * 
from goods 
where 1=1 
and goods_name like '商品名称_31501550617221'
and goods_type in ( '1' , '2' ,'3' ,'4' ,'5' ,'6' ,'7' ,'8' ,'9' )

执行时间 0.808s
执行计划:
id select_type  table   partitions type    possible_keys      key   key_len ref   rows     filtered   Extra  
1  SIMPLE       goods   Null       ALL     index_goods_name   Null  Null    Null  313590   100        Using where

结论：
对于列a 、b 的联合索引AB 如果where 中对b 列进行模糊查询，结果导致索引失效

```



对于a, b 列的联合索引AB
结论：使用联合索引查询可以将选择性低索引排在前面，如果where中只有 b 的列，数据库是无法使用索引查询的，这个时候可以在where 中构造 a 列的所有情况，拼接上 b 列条件，执行结果会使用索引，优化查询

