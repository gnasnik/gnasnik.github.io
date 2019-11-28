---
layout:     post
title:      "MySQL 开发规范"
subtitle:   ""
date:       2018-11-03 18:00:00
author:     "frank"
header-style: text
tags:
    - Mysql
    - 编程
---


## 基础规范

- 使用INNODB存储引擎
- 表字符集使用UTF8mb4
- 所有表都需要添加注释
- 单表数据量建议控制在500W以内
- 不在数据库中存储图片、文件等大数据
- 禁止在线上做数据库压力测试
- 禁止从测试、开发环境直连外网数据库
- 所有字段均定义为NOT NULL
- 使用UNSIGNED存储非负整数
- 程序应有捕获SQL异常的处理机制
- 尽量不使用select *


## 建表规范
1. 表达是与否概念的字段，必须使用 is_xxx的方式命名，数据类型是 unsigned tinyint（ 1表示是，0表示否）。
说明：任何字段如果为非负数，必须是 unsigned。
正例：表达逻辑删除的字段名 is_deleted，1 表示删除，0 表示未删除。
2. 表名、字段名必须使用小写字母或数字，禁止出现数字开头，禁止两个下划线中间只出现数字。
正例：getter_admin，task_config，level3_name
反例：GetterAdmin，taskConfig，level_3_name
3. 禁用保留字，如 desc、range、match、delayed等，请参考 MySQL官方保留字。
4. 主键索引名为 pk_字段名；唯一索引名为 uk_字段名；普通索引名则为 idx_字段名。
说明：pk_ 即 primary key；uk_ 即 unique key；idx_ 即 index的简称
5. 如果存储的字符串长度几乎相等，使用 char定长字符串类型。
6. varchar是可变长字符串，不预先分配存储空间，长度不要超过 5000，如果存储长度大于此值，定义字段类型为 text
7. __表必备三字段：id, create_time, update_time。__
说明：其中 id必为主键，类型为 int、单表时自增、步长为 1。create_time,update_time的类型均为 timestamp类型。
8. 表的命名最好是加上“业务名称_表的作用”。
正例：tiger_task  tiger_reader  mpp_config
9. 如果修改字段含义或对字段表示的状态追加时，需要及时更新字段注释
10. 字段允许适当冗余，以提高查询性能，但必须考虑数据一致。冗余字段应遵循：
* 1不是频繁修改的字段。
* 2不是 varchar超长字段，更不能是 text字段。
正例：商品类目名称使用频率高，字段长度短，名称基本一成不变，可在相关联的表中冗余存储类目名称，避免关联查询
11. 单表行数超过 500万行或者单表容量超过 2GB，才推荐进行分库分表。
说明：如果预计三年后的数据量根本达不到这个级别，请不要在创建表时就分库分表。
12. 合适的字符存储长度，不但节约数据库表空间、节约索引存储，更重要的是提升检索速度。

## 索引规范

1. 业务上具有唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。
说明：不要以为唯一索引影响了 insert速度，这个速度损耗可以忽略，但提高查找速度是明显的；另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必然有脏数据产生。
2. 超过三个表禁止 join。需要 join的字段，数据类型必须绝对一致；多表关联查询时，保证被关联的字段需要有索引。
说明：即使双表 join也要注意表索引、SQL性能。
3. 在 varchar字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据实际文本区分度决定索引长度即可。
说明：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为 20的索引，区分度会高达 90%以上，可以使用 count(distinct left(列名, 索引长度))/count(*)的区分度来确定。
4. 如果有 order by的场景，请注意利用索引的有序性。order by 最后的字段是组合索引的一部分，并且放在索引组合顺序的最后，避免出现 file_sort的情况，影响查询性能。
正例：where a=? and b=? order by c; 索引：a_b_c
反例：索引中有范围查找，那么索引有序性无法利用，如：WHERE a&gt;10 ORDER BY b; 索引a_b无法排序。

## SQL语句规范

1. 不要使用 count(列名)或 count(常量)来替代 count(*)，count(*)是 SQL92定义的标准统计行数的语法，跟数据库无关，跟 NULL和非 NULL无关。
说明：count(*)会统计值为 NULL的行，而 count(列名)不会统计此列为 NULL值的行。
2. count(distinctcol) 计算该列除 NULL之外的不重复行数，注意 count(distinctcol1, col2) 如果其中一列全为 NULL，那么即使另一列有不同的值，也返回为 0。
3. 当某一列的值全是 NULL时，count(col)的返回结果为 0，但 sum(col)的返回结果为NULL，因此使用 sum()时需注意 NPE问题。
4. 使用 ISNULL()来判断是否为 NULL值。注意：NULL与任何值的直接比较都为 NULL。
说明：
* 1 NULL&lt;&gt;NULL的返回结果是 NULL，而不是 false。
* 2 NULL=NULL的返回结果是 NULL，而不是 true。
* 3 NULL&lt;&gt;1的返回结果是 NULL，而不是 true。
5. 禁止使用存储过程，存储过程难以调试和扩展，更没有移植性。
6. in操作能避免则避免，若实在避免不了，需要仔细评估 in后边的集合元素数量，控制在 1000个之内。



## 例子
```
CREATE TABLE `t_contact` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `uid` int(11) NOT NULL COMMENT '用户id',
  `firstname` varchar(120) NOT NULL DEFAULT '' COMMENT '名',
  `lastname` varchar(120) NOT NULL DEFAULT '' COMMENT '姓`',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '状态，0=正常，1=解散',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_uid` (`uid`,`contact_uid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户通讯录表'

```