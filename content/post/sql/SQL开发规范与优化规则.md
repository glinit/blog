---
title: "SQL开发规范与优化规则"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["SQL", "优化"]
categories: ["SQL"]
author: "ChavinKing"
---

---



​	本文档说明优化技术主要考虑大数据环境SQL on Hadoop解决方案下的优化规则及开发规范，已尽力刨除RDBMS优化细节，且尽量不加杂关系代数专业术语。

​	根据多年的SQL优化经验，SQL优化可以从两个方向考虑：

* 其一优化人效，较好的SQL可读性，保证SQL语义简练完整，这也是最重要的；

* 其次优化SQL性能，保证代码执行质量。

  以下技术细节也是以这两个原则做为基础出发点论述SQL规范及SQL优化重点知识。

  

## 一 SQL优化技术 

### 1 查询重用 

* 查询结果重用 
  * 建立中间表 - 结果集重用 
  * 分区表 - 结果集重用 
  * from提前语法 - 结果集重用 
* 查询逻辑重用 
  * with子句 - 业务逻辑重用，提高SQL可读性 
* 查询计划重用 
  * OLAP环境 - 略 

### 2 查询重写 - 等价转换 

1. 简单查询优化 

   ```sql
   select ... 
   from tbl 
   where ... 
   ```

   优化规则：

   * 尽早执行选择（where过滤）、投影（select指定列）操作

2. 聚合查询优化 

   ```sql
   select 
   from tbl 
   where ... 
   group by ...  
   having ... 
   ```

   优化规则：

   * 符合简单查询规则；

   * 尽量提取having条件到where层过滤；

   * 粒度指标合并；一般首先选择一次聚合产出多个不同指标，避免同粒度指标的重复扫表；

3. 子查询优化 

   ```sql
   select ... 
   from (
       sub_query 
   ) ... 
   
   select ... 
   from tbl 
   where xxx in (
       sub_query 
   )
   ```

   优化规则：

   * 符合简单查询，聚合查询优化规则；

   * 子查询应只出现在from子句中，个别场景可以出现在where子句，如果很难满足该条件，应该重新审视数据结构设计的合理性；

   * 仅考虑非相关子查询（即子查询逻辑比较独立，可单独测试）开发，否则请审查数据结构设计的合理性；

   * 聚合子查询是子查询存在的标准模式；

   * 子查询内禁止使用order by子句；

   * 优化技巧：

     * 子查询合并：

       粒度相同指标可以考虑从同一个子查询产出（避免了多次扫表操作，减少io，提升了执行性能和可读性）；

     * 子查询展开：

       [not] in/[not] exists 子查询应尽量改写成join逻辑实现；

       单表扫描，非聚合子查询（即子查询符合简单SQL规则）应首先改写join逻辑实现；

     * 子查询消除：

       可以考虑使用窗口函数消除子查询；

4. join优化 

   ```sql
   select ... 
   from (
       sub_query 
   ) t1 left join (
       sub_query
   ) t2 on t1.xxx = t2.xxx and t2... 
   where t1... 
   and t2... 
   
   select ... 
   from tbl 
   where xxx in (
       sub_query
   )
   ```

   优化规则：

   * 外连接消除：如果外连接可以改写成内连接的SQL要改写成inner join逻辑；
   * join过程中需要时刻考虑两表关系：1:1 一对一 ；1:n 一对多 ；n:m 多对多 ，尽量避免局部笛卡尔积操作；
   * 使用left join语法替换子查询 ；
   * 如果是内连接，小表写在左侧；
   * join语法统一使用left join ，join 等模式进行设计；

   join示例具体执行计划参考：[案例参考](http://chavinking.github.io/post/sql/%E5%9F%BA%E4%BA%8EPGSQL%E5%B7%A6%E8%BF%9E%E6%8E%A5SQL%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%A7%A3%E6%9E%90/)

### 3 物理优化 

​	SQL语句查询代价在物理上基于CPU和IO的，在分布式系统领域还要加上网络IO开销。

​	基于此，物理优化的准则包括以下几种：

* 尽量减少回表次数；
* 尽早过滤数据集；
* 每一层子查询均需要指定查询列，降低内存使用量；
* 使用中间表、分区表进行增量加工；
* 优先使用parquet进行格式存储，归档数据首选orc+snappy进行数据存储；

​	

## 二 示例SQL解析 

​	以下每一个SQL例子用语言描述执行过程：

* 一个简单SQL 执行过程描述 

  ```sql
  select 
      e.deptno,
      count(1) as cnt,
      sum(sal) as sal 
  from public.employee e 
  where e.sal > 1000 
  group by e.deptno 
  having count(1) < 2 
  order by e.deptno 
  limit 10 
  ;
  ```

  from :  加载数据表 

  where :  应用过滤条件 

  【join】 : 按连接条件执行join 

  group by :  执行聚合逻辑 

  having : 针对聚合后的数据应用 过滤条件 

  select : 返回最终列 

  order by : 执行排序操作 

  limit : 执行limit 操作，返回最终结果。

* 一个join SQL 执行过程描述 

  ```sql
  select
      e.empno,
      e.ename,
      e.job,
      e.mgr,
      e.hiredate,
      e.sal,
      e.deptno,
      d.dname,
      d.loc
  from public.employee e 
  left join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.dname = 'ACCOUNTING' and d.deptno <= 30 
  where e.job = 'MANAGER' 
    and e.sal >= 1000 
    and d.deptno is null 
  ;
  ```

  加载左表、通过where子句左表的过滤条件过滤数据，结果集记为tmp1;

  加载右表、通过on子句右表的过滤条件过滤右表数据，结果集记为tmp2；

  临时结果tmp1与tmp2按照关联条件做左连接操作，tmp1未能关联到的tmp2数据，结构解tmp2列置null，join结果记为tmp3; 

  针对tmp3应用右表在where子句中的过滤条件，返回最终结果。

  

## 三 SQL开发规范 

1. 避免使用tab键格式化SQL（hive client 遇到tab键会报错）；

2. 禁止使用select * ，建议由select column ... 替代，不需要做绝对校验；

3. 多表join，且存在多inner join以及多left join场景，inner join 在最前，left join紧接其后；

   例如：

   ```sql
   select 
   
    t1.id as aid,
    t2.bname as bname,
    t3.cname as cname,
    t4.dname as dname,
    t5.ename as ename,
    t6.fname as fname 
   
   from tbl_a t1 
   join tbl_b t2 on t1.id = t2.aid 
   join tbl_c t3 on t1.id = t3.aid 
   
   left join tbl_d t4 on t1.id = t4.aid 
   left join tbl_e t5 on t1.id = t5.aid 
   left join tbl_f t6 on t1.id = t6.aid 
   ;
   ```

4. SQL开发select子句每一列最好统一带别名；

5. 建表语句都需要配置注释；建表语句子列格式需要上下对齐；

   例如：

   ```sql
   drop table if exists fact.fact_example_dtl;
   create table fact.fact_example_dtl(
   skey                    bigint             comment '代理键', 
   name                    string             comment '名字',
   age                     int                comment '年龄',
   sex                     int                comment '性别：1-男 2-女',
   create_date             string             comment '创建时间',
   update_date             string             comment '更新时间',
   etl_time                timestamp          comment '跑数时间'
   ) comment '签约事实表' 
   partitioned by (pdate string comment '分区日期:yyyyMMdd') 
   stored as parquet
   ;
   ```

6. 使用left join消除简单子查询；

   ```sql
   select ... 
   from (
       select * 
       from tbl_a 
       where age >= 10
   ) t1 left join (
       select * 
       from tbl_b 
       where sex = 1 
   ) t2 on t1.id = t2.aid 
   ;
   
   改写为 : 
   
   select ... 
   from tbl_a t1 
   left join tbl_b t2 on t1.id = t2.aid and t2.sex = 1 
   where t1.age >= 10 
   ;
   ```



