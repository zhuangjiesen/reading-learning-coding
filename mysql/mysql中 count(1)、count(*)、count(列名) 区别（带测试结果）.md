# mysql中 count(1)、count(*)、count(列名) 区别（带测试结果）

### 首先创建表
```

CREATE TABLE `biz_project` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(100) DEFAULT NULL COMMENT '项目名称',
  `purpose` varchar(100) DEFAULT NULL COMMENT '项目目的',
  `description` varchar(400) DEFAULT NULL COMMENT '描述信息',
  `industry_type` tinyint(4) DEFAULT NULL COMMENT '所属行业(1-采矿业，2-建筑业，3-文化娱乐，4-农业)',
  `work_place` varchar(100) DEFAULT NULL COMMENT '工作地点',
  `target_user` varchar(100) DEFAULT NULL COMMENT '目标用户',
  `budget` bigint(20) DEFAULT NULL COMMENT '项目预算',
  `online_platform_list` bigint(20) unsigned DEFAULT NULL COMMENT '上线平台',
  `gmt_publish` timestamp NULL DEFAULT NULL COMMENT '发布时间',
  `attach_url_list` varchar(512) DEFAULT NULL COMMENT '附件列表',
  `ar_template_type` tinyint(4) DEFAULT NULL COMMENT 'AR模板（0代表无，>=1代表各个模板）',
  `ar_appear_type` tinyint(4) DEFAULT NULL COMMENT 'AR内容出现方式（1-扫描物料，2-直接出现）',
  `share_type` tinyint(4) DEFAULT NULL COMMENT '是否需要社交分享（1-是，2-否）',
  `coupon_type` tinyint(4) DEFAULT NULL COMMENT '是否需要发优惠券*(1-需要，2-不需要)',
  `plan_case_type` tinyint(4) DEFAULT NULL COMMENT '是否已有策划案（1-是，2-否）',
  `status` int(10) DEFAULT '1' COMMENT '2-未审核，3-审核通过，10-审核不通过，4-选择AR服务商5-待确认合作，6-验收成果7-完成1-草稿状态',
  `consumer_id` bigint(20) unsigned DEFAULT NULL COMMENT '派单系统客户id',
  `gmt_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `gmt_create` timestamp NULL DEFAULT NULL COMMENT '创建时间',
  `reason` varchar(64) DEFAULT NULL COMMENT '项目驳回原因',
  `from_status` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  KEY `pd_consumer_id` (`consumer_id`) USING BTREE,
  KEY `idx_count` (`purpose`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=212307 DEFAULT CHARSET=utf8 COMMENT='事件表';

```


#### tips
```
1.普通的一张innodb的表
2.purpose 字段是个普通索引
3.已经通过脚本插入了21万条记录

```


### 测试步骤以及结果
```

-- 例1 无条件
explain select count(1) from biz_project 
-- 结果
1	SIMPLE	biz_project	index		pd_consumer_id	9		210358	Using index

explain select count(*) from biz_project 
-- 结果
1	SIMPLE	biz_project	index		pd_consumer_id	9		210358	Using index

-- 例2 带where跳转 (普通索引)
explain select count(1) from biz_project where ar_template_type = '1'
-- 结果
1	SIMPLE	biz_project	ALL					210358	Using where

explain select count(*) from biz_project where ar_template_type = '1'
-- 结果
1	SIMPLE	biz_project	ALL					210358	Using where


-- 例3 索引
explain select count(1) from biz_project where purpose like 'node自定义%'
-- 结果
1	SIMPLE	biz_project	range	idx_count	idx_count	303		105211	Using where; Using index

explain select count(*) from biz_project where purpose like 'node自定义%'
-- 结果
1	SIMPLE	biz_project	range	idx_count	idx_count	303		105211	Using where; Using index


explain select count(online_platform_list) from biz_project
-- 例1：
-- 数据(部分)：
-- id    coupon_type online_platform_list
-- 2312  0           null
select count(1) from biz_project where 1=1 and id = 2312
-- 结果
1
--
select count(coupon_type) from biz_project where 1=1 and id = 2312
-- 结果
1
--
select count(online_platform_list) from biz_project where 1=1 and id = 2312
-- 结果
0


```

### 结论

```
-- 结论
1.count(1) 和 count(*) 基本没有区别
2.count(列名) 会忽略null 数据而''(空字符串) 或者0 会被计数

```


