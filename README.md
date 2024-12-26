
本文分享自华为云社区[《GaussDB(DWS) 谓词列analyze揭秘》](https://github.com)，作者：SmithCoder。


# 1\. 前言


适用版本：【9\.1\.0\.100（及以上）】


​当前GaussDB(DWS)中存在手动analyze，查询触发的动态analyze，以及后台线程的轮询analyze三种触发形式，其中动态analyze又分为light模式和normal模式，light模式是基于内存存储统计信息，normal模式是存储在系统表里，前者较为轻量，对目标表仅加一级锁。这些analyze默认都是对表全列进行采样，尤其对于大宽表analyze，耗时会比较久，其实很多时候计划生成所需的统计信息列只占一部分，如果只对这部分列进行采样计算，将会大大节省analyze的时间。因此GaussDB(DWS)引入了谓词列analyze。


# 2\.原理介绍


​analyze的时候主要耗时在采样数据，对于非定长列如varchar，其统计信息的计算也耗时，耗时肯定随着表的列数增加而变长。谓词列analyze会在查询阶段对谓词列进行识别收集，当触发动态analyze，会只选择采样谓词列，手动谓词列analyze也会只采样谓词列部分。


![](https://img2024.cnblogs.com/blog/2030258/202412/2030258-20241225172041263-145989174.png)


谓词列通指于 WHERE 条件，join条件，group by中涉及到的列，更广义的是指所有需要用于计划生成需要统计信息列的列。


识别到的谓词列包含：


* 条件谓词列


如比较（\=,\>,\<），like, between,is,in, having，join on,MERGE on后面的列


* 排序分组列


如group by， order by, distinct, over partition/order by列，union /minus列


* 子查询引用列


等值条件列为一个子查询，子查询的对应的输出列；


cte中的一个列被外层引用为谓词列；


视图中的列被外层应用为谓词列；


agg列


* 索引列


列建立了索引，相应的列也会被收集识别


* 分布列


如果表建立了hash分别列，分布列也收集识别


# 3\.使用介绍


下面分别介绍谓词列analyze支持两种形式：


1. 动态采样谓词列analyze
2. 手动谓词列analyze


## 3\.1谓词列的guc控制


谓词列analyze的开启由guc参数**analyze\_predicate\_column\_threshold**控制


**参数说明**：控制是否开启谓词列ANALYZE及限定支持的最小列数。该参数仅9\.1\.0\.100及以上集群版本支持。


**参数类型**：SIGHUP


**取值范围**：整型，0～10000


* 0表示关闭谓词列ANALYZE，不会收集谓词列以及对谓词列进行ANALYZE。
* 大于0表示开启谓词列收集功能，且仅对列数大于等于此值的表进行谓词列ANALYZE。


**默认值**：10


## 3\.2\.动态采样谓词列analyze


前提：动态采样谓词列analyze只支持light模式


查询的时候会进行谓词列的识别，如果有新的谓词列或者修改计数达到，则会触发动态采样，会把新识别的和已有的谓词列进行采样计算 统计信息。


下面是按照analyze\_predicate\_column\_threshold\=1举例，计划中RunTime Analyze Information部分会打印出analyze的表和列信息。



```
create table t1(a int, b int, c int);
create table t2(a int, b int, c int);
set enable_fast_query_shipping=off;
set autoanalyze_mode=light;
-- 首次会收集所有列的统计信息
test=# explain select * from t1;
                                                            QUERY PLAN                                                            
-------------------------------------------------------------------------------------------------------------------------------
  id |          operation           | E-rows | E-memory | E-width | E-costs 
 ----+------------------------------+--------+----------+---------+---------
   1 | ->  Streaming (type: GATHER) |     20 |          |      12 | 16.10   
   2 |    ->  Seq Scan on t1        |     20 | 1MB      |      12 | 10.10   
 
                                                   RunTime Analyze Information                                                   
 ------------------------------------------------------------------------------------------------------------------------------
       light runtime analyze on "public.t1" times:9.245ms, stats:sync, change:0(0 in xact), alive:0.000000, threshold:50.000000
 
   ====== Query Summary =====   
 -------------------------------
 System available mem: 4710400KB
 Query Max mem: 4710400KB
 Query estimated mem: 1024KB
 
 insert into t1 select generate_series(1, 100), 1;
 
 -- 修改计数达到，收集到了a,b两个谓词列，触发动态采样
 test=# explain select * from t1 where a=b;
                                                                QUERY PLAN                                                                 
--------------------------------------------------------------------------------------------------------------------------------
  id |          operation           | E-rows | E-memory | E-width | E-costs 
 ----+------------------------------+--------+----------+---------+---------
   1 | ->  Streaming (type: GATHER) |      1 |          |      12 | 7.62    
   2 |    ->  Seq Scan on t1        |      1 | 1MB      |      12 | 1.62    
 
                                                        RunTime Analyze Information                                                       
 -------------------------------------------------------------------------------------------------------------------------------
     light runtime analyze on "public.t1(a,b)" times:9.774ms, stats:sync, change:100(0 in xact), alive:100.000000, threshold:50
 
 Predicate Information (identified by plan id)
 ---------------------------------------------
   2 --Seq Scan on t1
         Filter: (a = b)
 
   ====== Query Summary =====   
 -------------------------------
 System available mem: 4710400KB
 Query Max mem: 4710400KB
 Query estimated mem: 1024KB
(19 rows)

-- 修改计数达到后查询，查询下面的语句，会分别收集表t1,t2对应的谓词列
test=# explain select t1.* from t1 join t2 on t1.b=t2.c where t1.a=t2.a;
                                                                 QUERY PLAN                                                     
--------------------------------------------------------------------------------------------------------------------------------
  id |              operation               | E-rows | E-memory | E-width | E-costs 
 ----+--------------------------------------+--------+----------+---------+---------
   1 | ->  Streaming (type: GATHER)         |      2 |          |      12 | 21.55   
   2 |    ->  Hash Join (3,5)               |      2 | 1MB      |      12 | 15.55   
   3 |       ->  Streaming(type: BROADCAST) |    400 | 2MB      |       8 | 10.93   
   4 |          ->  Seq Scan on t2          |    200 | 1MB      |       8 | 2.00    
   5 |       ->  Hash                       |    200 | 16MB     |      12 | 2.00    
   6 |          ->  Seq Scan on t1          |    200 | 1MB      |      12 | 2.00    
 
                                                        RunTime Analyze Information                                                        
 -------------------------------------------------------------------------------------------------------------------------------
     light runtime analyze on "public.t1(a,b)" times:11.031ms, stats:sync, change:100(0 in xact), alive:200.000000, threshold:60
     light runtime analyze on "public.t2(a,c)" times:8.571ms, stats:sync, change:100(0 in xact), alive:200.000000, threshold:60
 
    Predicate Information (identified by plan id)    
 ----------------------------------------------------
   2 --Hash Join (3,5)
         Hash Cond: ((t2.c = t1.b) AND (t2.a = t1.a))
 
   ====== Query Summary =====   
 -------------------------------
 System available mem: 4710400KB
 Query Max mem: 4710400KB
 Query estimated mem: 4388KB

-- 遇到新的谓词列且该列没有统计信息也会触发
 test=# explain select * from t1 where c=1;
                                                                 QUERY PLAN                                                                 
--------------------------------------------------------------------------------------------------------------------------------
  id |          operation           | E-rows | E-memory | E-width | E-costs 
 ----+------------------------------+--------+----------+---------+---------
   1 | ->  Streaming (type: GATHER) |      1 |          |      12 | 8.25    
   2 |    ->  Seq Scan on t1        |      1 | 1MB      |      12 | 2.25    
 
                                                        RunTime Analyze Information                                                        
 -------------------------------------------------------------------------------------------------------------------------------
    light runtime analyze on "public.t1(a,b,c)" times:10.063ms, stats:sync, change:0(0 in xact), alive:200.000000, threshold:70
```

## 3\.3手动谓词列


语法：**analyze (predicate) tablename;**


执行后，只会对当前为止收集到的谓词列进行采样，最后在查询业务稳定后使用；


## 3\.4谓词列管理


* **查询谓词列**


通过函数**pg\_stat\_get\_predicate\_columns**可以查询当前表有哪些谓词列



```
test=# select pr.attnum,pa.attname from pg_catalog.pg_stat_get_predicate_columns('t1'::regclass) pr left join pg_attribute pa on pa.attrelid='t1'::regclass and pa.attnum = pr.attnum;
 attnum | attname 
--------+---------
      1 | a
      2 | b
(2 rows)
```

* **清空谓词列**


通过函数select \* from pg\_catalog.pg\_stat\_get\_predicate\_columns(‘t1\_3’::regclass);可以清空表的谓词列，一般用于谓词列太多或者过期的情况去清空重建


# 4\.总结


9\.1\.0\.100版本中autoanalyze(light模式)和谓词列默认是开启的，用户无需感知。对于频繁大量更新和查询场景，此前的动态采样可能会耗时，影响查询业务，新增谓词列后会减少analyze的耗时，特别对大宽表场景耗时优化是可观的。


# 5\. 参考文档


一文读懂analyze使用【这次高斯不是数学家】 [https://bbs.huaweicloud.com/blogs/354294](https://github.com)


一文读懂autoanalyze使用【这次高斯不是数学家】 [https://bbs.huaweicloud.com/blogs/354298](https://github.com):[slowerssr加速器](https://slowerss.com)


华为开发者空间，汇聚鸿蒙、昇腾、鲲鹏、GaussDB、欧拉等各项根技术的开发资源及工具，致力于为每位开发者提供一台云主机、一套开发工具及云上存储空间，让开发者基于华为根生态创新。[点击链接](https://github.com)，免费领取您的专属云主机。


 


[**点击关注，第一时间了解华为云新鲜技术\~**](https://github.com)


 


