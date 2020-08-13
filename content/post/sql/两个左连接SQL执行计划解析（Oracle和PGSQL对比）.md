---
title: "两个左连接SQL执行计划解析（Oracle和PGSQL对比）"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["SQL", "优化"]
categories: ["SQL"]
author: "ChavinKing"
---

___



1、SQL示例1：

```sql
select * 
from (
select * from tmp1 where c >= 1
) t1 left join (
select * from tmp2 where b < 30
) t2 on t1.a = t2.a 
and t2.d > 1 and t1.e >= 2 
where t1.b < 50 
;
```



​	Oracle执行结果：

```sql
SQL> select * 
from (
select * from tmp1 where c >= 1
) t1 left join (
select * from tmp2 where b < 30
) t2 on t1.a = t2.a 
and t2.d > 1 and t1.e >= 2 
where t1.b < 50 
;

	 A	    B	       C	  E	     A		B	   D	      E
---------- ---------- ---------- ---------- ---------- ---------- ---------- ----------
	 2	   20	       2	  2	     2	       20	   2	      2
	 4	   40	       4	  4
	 3	   30	       3	  3
	 1	   10	       1	  1


Execution Plan
----------------------------------------------------------
Plan hash value: 2592321047

---------------------------------------------------------------------------
| Id  | Operation	   | Name | Rows  | Bytes | Cost (%CPU)| Time	  |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |	  |	4 |   416 |	7  (15)| 00:00:01 |
|*  1 |  HASH JOIN OUTER   |	  |	4 |   416 |	7  (15)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL| TMP1 |	4 |   208 |	3   (0)| 00:00:01 |
|*  3 |   TABLE ACCESS FULL| TMP2 |	1 |    52 |	3   (0)| 00:00:01 |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("TMP1"."A"="TMP2"."A"(+))
       filter("TMP1"."E">=CASE	WHEN ("TMP2"."A"(+) IS NOT NULL) THEN 2
	      ELSE 2 END )
   2 - filter("TMP1"."B"<50 AND "C">=1)
   3 - filter("TMP2"."D"(+)>1 AND "B"(+)<30)

Note
-----
   - dynamic sampling used for this statement (level=2)


Statistics
----------------------------------------------------------
	  0  recursive calls
	  0  db block gets
	  7  consistent gets
	  0  physical reads
	  0  redo size
       1082  bytes sent via SQL*Net to client
	524  bytes received via SQL*Net from client
	  2  SQL*Net roundtrips to/from client
	  0  sorts (memory)
	  0  sorts (disk)
	  4  rows processed
```



​	PostgreSQL示例：

```sql
postgres=# explain analyze select * 
postgres-# from (
postgres(# select * from tmp1 where c >= 1
postgres(# ) t1 left join (
postgres(# select * from tmp2 where b < 30
postgres(# ) t2 on t1.a = t2.a 
postgres-# and t2.d > 1 and t1.e >= 2 
postgres-# where t1.b < 50 
postgres-# ;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Hash Left Join  (cost=34.90..80.00 rows=181 width=32) (actual time=0.021..0.035 rows=4 loops=1)
   Hash Cond: ("outer".a = "inner".a)
   Join Filter: ("outer".e >= 2)
   ->  Seq Scan on tmp1  (cost=0.00..34.45 rows=181 width=16) (actual time=0.006..0.011 rows=4 loops=1)
         Filter: ((c >= 1) AND (b < 50))
   ->  Hash  (cost=34.45..34.45 rows=181 width=16) (actual time=0.007..0.007 rows=1 loops=1)
         ->  Seq Scan on tmp2  (cost=0.00..34.45 rows=181 width=16) (actual time=0.002..0.003 rows=1 loops=1)
               Filter: ((b < 30) AND (d > 1))
 Total runtime: 0.063 ms
(9 rows)

```



​	执行计划分析：

```wik
1) 全表扫描左表TMP1，同时根据TMP1表子查询条件"C">=1和where过滤条件"T1"."B"<50联合过滤，即filter("TMP1"."B"<50 AND "C">=1)，计算结果临时表记为tmp1；
2) 全表扫描右表TMP2，同时根据TMP2表子查询条件"B"(+)<30和on子句"T2"."D"(+)>1联合过滤，即filter("TMP2"."D"(+)>1 AND "B"(+)<30)，计算结果临时表记为tmp2；
3) 左表TMP1及右表TMP2处理后临时表tmp1和tmp2通过access("TMP1"."A"="TMP2"."A"(+))连接条件进行Hash Left Join操作，左临时表结果集全量返回，右表不匹配行置为null，返回结果临时表记为tmp3；
4) 返回结果集。
```



2、SQL示例2：

```sql
select * 
from (
select * from tmp1 where c >= 1
) t1 left join (
select * from tmp2 where b < 30
) t2 on t1.a = t2.a 
and t2.d > 1 and t1.e >= 2 
where t1.b < 50 and t2.e <= 3 
;
```



​	Oracle示例：

```sql
SQL> select * 
from (
select * from tmp1 where c >= 1
) t1 left join (
select * from tmp2 where b < 30
) t2 on t1.a = t2.a 
and t2.d > 1 and t1.e >= 2 
where t1.b < 50 and t2.e <= 3 
;

	 A	    B	       C	  E	     A		B	   D	      E
---------- ---------- ---------- ---------- ---------- ---------- ---------- ----------
	 2	   20	       2	  2	     2	       20	   2	      2


Execution Plan
----------------------------------------------------------
Plan hash value: 1630095649

---------------------------------------------------------------------------
| Id  | Operation	   | Name | Rows  | Bytes | Cost (%CPU)| Time	  |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |	  |	1 |   104 |	7  (15)| 00:00:01 |
|*  1 |  HASH JOIN	   |	  |	1 |   104 |	7  (15)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL| TMP2 |	1 |    52 |	3   (0)| 00:00:01 |
|*  3 |   TABLE ACCESS FULL| TMP1 |	3 |   156 |	3   (0)| 00:00:01 |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("TMP1"."A"="TMP2"."A")
   2 - filter("TMP2"."E"<=3 AND "TMP2"."D">1 AND "B"<30)
   3 - filter("TMP1"."B"<50 AND "TMP1"."E">=2 AND "C">=1)

Note
-----
   - dynamic sampling used for this statement (level=2)


Statistics
----------------------------------------------------------
	  9  recursive calls
	  0  db block gets
	 15  consistent gets
	  0  physical reads
	  0  redo size
	981  bytes sent via SQL*Net to client
	524  bytes received via SQL*Net from client
	  2  SQL*Net roundtrips to/from client
	  0  sorts (memory)
	  0  sorts (disk)
	  1  rows processed

```



​	PostgreSQL示例：

```sql
postgres=# select * 
postgres-# from (
postgres(# select * from tmp1 where c >= 1
postgres(# ) t1 left join (
postgres(# select * from tmp2 where b < 30
postgres(# ) t2 on t1.a = t2.a 
postgres-# and t2.d > 1 and t1.e >= 2 
postgres-# where t1.b < 50 and t2.e <= 3 
postgres-# ;
 a | b  | c | e | a | b  | d | e 
---+----+---+---+---+----+---+---
 2 | 20 | 2 | 2 | 2 | 20 | 2 | 2
(1 row)

postgres=# explain analyze select * 
postgres-# from (
postgres(# select * from tmp1 where c >= 1
postgres(# ) t1 left join (
postgres(# select * from tmp2 where b < 30
postgres(# ) t2 on t1.a = t2.a 
postgres-# and t2.d > 1 and t1.e >= 2 
postgres-# where t1.b < 50 and t2.e <= 3 
postgres-# ;
                                                 QUERY PLAN                                                  
-------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=38.68..78.43 rows=18 width=32) (actual time=0.033..0.041 rows=1 loops=1)
   Hash Cond: ("outer".a = "inner".a)
   ->  Seq Scan on tmp1  (cost=0.00..38.53 rows=60 width=16) (actual time=0.007..0.011 rows=3 loops=1)
         Filter: ((c >= 1) AND (e >= 2) AND (b < 50))
   ->  Hash  (cost=38.53..38.53 rows=60 width=16) (actual time=0.008..0.008 rows=1 loops=1)
         ->  Seq Scan on tmp2  (cost=0.00..38.53 rows=60 width=16) (actual time=0.003..0.005 rows=1 loops=1)
               Filter: ((b < 30) AND (d > 1) AND (e <= 3))
 Total runtime: 0.070 ms
(8 rows)
```



​	执行计划分析：

```wik
1) 全表扫描左表TMP2，同时根据TMP2表子查询条件"B"<30和where过滤条件"TMP2"."E"<=3及ON子句过滤条件"TMP2"."D">1联合过滤，即filter("TMP2"."E"<=3 AND "TMP2"."D">1 AND "B"<30)，计算结果临时表记为tmp1；
2) 全表扫描右表TMP1，同时根据TMP1表子查询条件"C">=1和where子句过滤条件"TMP1"."B"<50及ON子句"TMP1"."E">=2联合过滤，即filter("TMP1"."B"<50 AND "TMP1"."E">=2 AND "C">=1)，计算结果临时表记为tmp2；
3) 临时表tmp1和tmp2通过access("TMP1"."A"="TMP2"."A")连接条件进行Hash Join连接操作（此处left join写法已经被转换为内链接），返回匹配结果临时表记为tmp3；
4) 返回结果集。

```

