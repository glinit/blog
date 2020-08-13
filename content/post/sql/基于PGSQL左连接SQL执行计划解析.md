---
title: "基于PGSQL左连接SQL执行计划解析"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["SQL", "优化"]
categories: ["SQL"]
author: "ChavinKing"
---

___

# 一 示例数据 

```sql
-- table employee 
drop table if exists employee;
create table employee(
  empno    int,
  ename    VARCHAR(10),
  job      VARCHAR(9),
  mgr      int,
  hiredate DATE,
  sal      decimal(7,2),
  comm     decimal(7,2),
  deptno   int
)
;

insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7888, 'CHAVIN', 'MANAGER', 7839, to_date('15-03-1990', 'dd-mm-yyyy'), 3000, 10000, 10);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7889, 'NOPE', 'MANAGER', 7839, to_date('12-02-1991', 'dd-mm-yyyy'), 3500, 10000, 20);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7369, 'SMITH', 'CLERK', 7902, to_date('17-12-1980', 'dd-mm-yyyy'), 800, null, 20);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7499, 'ALLEN', 'SALESMAN', 7698, to_date('20-02-1981', 'dd-mm-yyyy'), 1600, 300, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7521, 'WARD', 'SALESMAN', 7698, to_date('22-02-1981', 'dd-mm-yyyy'), 1250, 500, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7566, 'JONES', 'MANAGER', 7839, to_date('02-04-1981', 'dd-mm-yyyy'), 2975, null, 40);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7654, 'MARTIN', 'SALESMAN', 7698, to_date('28-09-1981', 'dd-mm-yyyy'), 1250, 1400, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7698, 'BLAKE', 'MANAGER', 7839, to_date('01-05-1981', 'dd-mm-yyyy'), 2850, null, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7782, 'CLARK', 'MANAGER', 7839, to_date('09-06-1981', 'dd-mm-yyyy'), 2450, null, 10);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7788, 'SCOTT', 'ANALYST', 7566, to_date('19-04-1987', 'dd-mm-yyyy'), 3000, null, 20);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7839, 'KING', 'PRESIDENT', null, to_date('17-11-1981', 'dd-mm-yyyy'), 5000, null, 10);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7844, 'TURNER', 'SALESMAN', 7698, to_date('08-09-1981', 'dd-mm-yyyy'), 1500, 0, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7876, 'ADAMS', 'CLERK', 7788, to_date('23-05-1987', 'dd-mm-yyyy'), 1100, null, 20);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7900, 'JAMES', 'CLERK', 7698, to_date('03-12-1981', 'dd-mm-yyyy'), 950, null, 30);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7902, 'FORD', 'ANALYST', 7566, to_date('03-12-1981', 'dd-mm-yyyy'), 3000, null, 20);
insert into employee (empno, ename, job, mgr, hiredate, sal, comm, deptno)values (7934, 'MILLER', 'CLERK', 7782, to_date('23-01-1982', 'dd-mm-yyyy'), 1300, null, 10);

```



```sql
-- table department 
drop table if exists department;
create table department(
  deptno int,
  dname  VARCHAR(14),
  loc    VARCHAR(13)
)
;

insert into department (deptno, dname, loc)values (10, 'ACCOUNTING', 'NEW YORK');
insert into department (deptno, dname, loc)values (20, 'RESEARCH', 'NEW YORK');
insert into department (deptno, dname, loc)values (30, 'SALES', 'CHICAGO');
insert into department (deptno, dname, loc)values (40, 'OPERATIONS', 'BOSTON');

```



# 二 执行计划解析

## 1、SQL示例1

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
from (
	select * from public.employee where sal >= 1000
) e left join (
	select * from public.department where deptno <= 30
) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
where e.job = 'MANAGER' 
;

```



​	执行计划结果：

```sql
postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from (
postgres(# select * from public.employee where sal >= 1000
postgres(# ) e left join (
postgres(# select * from public.department where deptno <= 30
postgres(# ) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
postgres-# where e.job = 'MANAGER' 
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7889 | NOPE   | MANAGER | 7839 | 1991-02-12 | 3500.00 |     20 | RESEARCH   | NEW YORK
  7566 | JONES  | MANAGER | 7839 | 1981-04-02 | 2975.00 |     40 |            | 
  7698 | BLAKE  | MANAGER | 7839 | 1981-05-01 | 2850.00 |     30 |            | 
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(5 rows)
-- --------------------------------------------------------------------------------------------------------------------

postgres=# explain analyze select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from (
postgres(# select * from public.employee where sal >= 1000
postgres(# ) e left join (
postgres(# select * from public.department where deptno <= 30
postgres(# ) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
postgres-# where e.job = 'MANAGER' 
postgres-# ;
                                                 QUERY PLAN                                                 
------------------------------------------------------------------------------------------------------------
 Nested Loop Left Join  (cost=0.00..38.16 rows=1 width=194) (actual time=0.080..0.098 rows=5 loops=1)
   Join Filter: (employee.deptno = department.deptno)
   Rows Removed by Join Filter: 7
   ->  Seq Scan on employee  (cost=0.00..18.25 rows=1 width=104) (actual time=0.069..0.075 rows=5 loops=1)
         Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))
         Rows Removed by Filter: 11
   ->  Seq Scan on department  (cost=0.00..19.90 rows=1 width=94) (actual time=0.002..0.002 rows=2 loops=5)
         Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text))
         Rows Removed by Filter: 2
 Planning time: 0.969 ms
 Execution time: 0.151 ms
(11 rows)
```



​	执行计划解释：

```wik
1) 全表扫描左表employee，同时根据employee表子查询条件(sal >= '1000'::numeric)和where过滤条件(job)::text = 'MANAGER'::text)联合过滤，即Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))，计算结果临时表记为tmp1;
2) 全表扫描右表department，同时根据department表子查询条件(deptno <= 30)和on子句(loc)::text = 'NEW YORK'::text)联合过滤，即Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text))，计算结果临时表记为tmp2;
3) 左表employee及右表department处理后临时表tmp1和tmp2通过Join Filter: (e.deptno = d.deptno)连接条件进行Nested Loop Left Join(嵌套循环左连接)操作，左临时表结果集全量返回，右表不匹配行置为null，返回结果临时表记为tmp;
4) 返回结果集。
```



​	标准化SQL写法：

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
left join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.deptno <= 30 
where e.job = 'MANAGER' and e.sal >= 1000 
;

```



```sql
postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# left join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.deptno <= 30 
postgres-# where e.job = 'MANAGER' and e.sal >= 1000 
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7889 | NOPE   | MANAGER | 7839 | 1991-02-12 | 3500.00 |     20 | RESEARCH   | NEW YORK
  7566 | JONES  | MANAGER | 7839 | 1981-04-02 | 2975.00 |     40 |            | 
  7698 | BLAKE  | MANAGER | 7839 | 1981-05-01 | 2850.00 |     30 |            | 
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(5 rows)
-- --------------------------------------------------------------------------------------------------------------------

postgres=# explain analyze select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# left join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.deptno <= 30 
postgres-# where e.job = 'MANAGER' and e.sal >= 1000 
postgres-# ;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Nested Loop Left Join  (cost=0.00..38.16 rows=1 width=194) (actual time=0.012..0.031 rows=5 loops=1)
   Join Filter: (e.deptno = d.deptno)
   Rows Removed by Join Filter: 7
   ->  Seq Scan on employee e  (cost=0.00..18.25 rows=1 width=104) (actual time=0.008..0.016 rows=5 loops=1)
         Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))
         Rows Removed by Filter: 11
   ->  Seq Scan on department d  (cost=0.00..19.90 rows=1 width=94) (actual time=0.001..0.002 rows=2 loops=5)
         Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text))
         Rows Removed by Filter: 2
 Planning time: 0.093 ms
 Execution time: 0.061 ms
(11 rows)
```



​	改写后的执行计划解释同上，改写后SQL可读性大大增加。



## 2、SQL示例2

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
from (
    select * from public.employee where sal >= 1000
) e left join (
    select * from public.department where deptno <= 30
) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
where e.job = 'MANAGER' and d.dname = 'ACCOUNTING'
;

```



​	执行计划结果：

```sql
postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from (
postgres(#     select * from public.employee where sal >= 1000
postgres(# ) e left join (
postgres(#     select * from public.department where deptno <= 30
postgres(# ) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
postgres-# where e.job = 'MANAGER' and d.dname = 'ACCOUNTING'
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(2 rows)
-- --------------------------------------------------------------------------------------------------------------------

postgres=# explain analyze select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from (
postgres(#     select * from public.employee where sal >= 1000
postgres(# ) e left join (
postgres(#     select * from public.department where deptno <= 30
postgres(# ) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
postgres-# where e.job = 'MANAGER' and d.dname = 'ACCOUNTING'
postgres-# ;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..39.81 rows=1 width=194) (actual time=0.017..0.052 rows=2 loops=1)
   Join Filter: (employee.deptno = department.deptno)
   Rows Removed by Join Filter: 3
   ->  Seq Scan on employee  (cost=0.00..18.25 rows=1 width=104) (actual time=0.010..0.017 rows=5 loops=1)
         Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))
         Rows Removed by Filter: 11
   ->  Seq Scan on department  (cost=0.00..21.55 rows=1 width=94) (actual time=0.002..0.005 rows=1 loops=5)
         Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text) AND ((dname)::text = 'ACCOUNTING'::text))
         Rows Removed by Filter: 3
 Planning time: 0.242 ms
 Execution time: 0.105 ms
(11 rows)
         
-- --------------------------------------------------------------------------------------------------------------------
postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from (
postgres(#     select * from public.employee where sal >= 1000
postgres(# ) e join (
postgres(#     select * from public.department where deptno <= 30
postgres(# ) d on e.deptno = d.deptno and d.loc = 'NEW YORK' 
postgres-# where e.job = 'MANAGER' and d.dname = 'ACCOUNTING'
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(2 rows)
```



​	执行计划解释：此写法将left join语义自动优化为join语义

```wik
1) 全表扫描左表employee，同时根据employee表子查询条件(sal >= '1000'::numeric)和where过滤条件(job)::text = 'MANAGER'::text)联合过滤，即Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))，计算结果临时表记为tmp1；
2) 全表扫描右表department，同时根据department表子查询条件(deptno <= 30)和on子句过滤条件((loc)::text = 'NEW YORK'::text)以及where子句过滤条件((dname)::text = 'ACCOUNTING'::text)联合过滤，即Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text) AND ((dname)::text = 'ACCOUNTING'::text))，计算结果临时表记为tmp2；
3) 临时表tmp1和tmp2通过Join Filter: (employee.deptno = department.deptno)连接条件进行Nested Loop（嵌套循环连接）操作（此处left join写法已经被转换为内链接），返回匹配结果临时表记为tmp3；
4) 返回结果集。

```



​	标准化SQL写法：

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
join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.dname = 'ACCOUNTING' and d.deptno <= 30 
where e.job = 'MANAGER' and e.sal >= 1000 
;

	或者 

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
join public.department d on e.deptno = d.deptno 
where e.job = 'MANAGER' 
  and e.sal >= 1000 
  and d.loc = 'NEW YORK' 
  and d.dname = 'ACCOUNTING' 
  and d.deptno <= 30 
;
```



​	执行计划：

```sql
postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.dname = 'ACCOUNTING' and d.deptno <= 30 
postgres-# where e.job = 'MANAGER' and e.sal >= 1000 
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(2 rows)

postgres=# 
postgres=# explain analyze select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# join public.department d on e.deptno = d.deptno and d.loc = 'NEW YORK' and d.dname = 'ACCOUNTING' and d.deptno <= 30 
postgres-# where e.job = 'MANAGER' and e.sal >= 1000 
postgres-# ;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..39.81 rows=1 width=194) (actual time=0.012..0.029 rows=2 loops=1)
   Join Filter: (e.deptno = d.deptno)
   Rows Removed by Join Filter: 3
   ->  Seq Scan on employee e  (cost=0.00..18.25 rows=1 width=104) (actual time=0.007..0.013 rows=5 loops=1)
         Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))
         Rows Removed by Filter: 11
   ->  Seq Scan on department d  (cost=0.00..21.55 rows=1 width=94) (actual time=0.001..0.002 rows=1 loops=5)
         Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text) AND ((dname)::text = 'ACCOUNTING'::text))
         Rows Removed by Filter: 3
 Planning time: 0.094 ms
 Execution time: 0.085 ms
(11 rows)

postgres=# select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# join public.department d on e.deptno = d.deptno 
postgres-# where e.job = 'MANAGER' 
postgres-#   and e.sal >= 1000 
postgres-#   and d.loc = 'NEW YORK' 
postgres-#   and d.dname = 'ACCOUNTING' 
postgres-#   and d.deptno <= 30 
postgres-# ;
 empno | ename  |   job   | mgr  |  hiredate  |   sal   | deptno |   dname    |   loc    
-------+--------+---------+------+------------+---------+--------+------------+----------
  7888 | CHAVIN | MANAGER | 7839 | 1990-03-15 | 3000.00 |     10 | ACCOUNTING | NEW YORK
  7782 | CLARK  | MANAGER | 7839 | 1981-06-09 | 2450.00 |     10 | ACCOUNTING | NEW YORK
(2 rows)

postgres=# explain analyze select
postgres-#     e.empno,
postgres-#     e.ename,
postgres-#     e.job,
postgres-#     e.mgr,
postgres-#     e.hiredate,
postgres-#     e.sal,
postgres-#     e.deptno,
postgres-#     d.dname,
postgres-#     d.loc
postgres-# from public.employee e 
postgres-# join public.department d on e.deptno = d.deptno 
postgres-# where e.job = 'MANAGER' 
postgres-#   and e.sal >= 1000 
postgres-#   and d.loc = 'NEW YORK' 
postgres-#   and d.dname = 'ACCOUNTING' 
postgres-#   and d.deptno <= 30 
postgres-# ;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..39.81 rows=1 width=194) (actual time=0.492..0.517 rows=2 loops=1)
   Join Filter: (e.deptno = d.deptno)
   Rows Removed by Join Filter: 3
   ->  Seq Scan on employee e  (cost=0.00..18.25 rows=1 width=104) (actual time=0.007..0.015 rows=5 loops=1)
         Filter: ((sal >= '1000'::numeric) AND ((job)::text = 'MANAGER'::text))
         Rows Removed by Filter: 11
   ->  Seq Scan on department d  (cost=0.00..21.55 rows=1 width=94) (actual time=0.097..0.099 rows=1 loops=5)
         Filter: ((deptno <= 30) AND ((loc)::text = 'NEW YORK'::text) AND ((dname)::text = 'ACCOUNTING'::text))
         Rows Removed by Filter: 3
 Planning time: 0.093 ms
 Execution time: 0.541 ms
(11 rows)
```



​	开发规范总结：

```wiki
- SQL join操作，应该首先保证过滤无效数据，尽量在join之前保证结果集最小化；
- 连接操作过滤：
	- 左表 首先以 子查询 条件 以及 where 条件过滤数据 
	- 右表 首先以 子查询 条件 以及 on子句条件过滤数据 
- 右表 非null 过滤条件出现在where子句，SQL可改写为inner join；右表null条件出现在where子句 语义同 not in；
- join操作 子查询能改写成单层join语义的尽量都改写成join语义，一般仅在子查询应用聚合语义才使用子查询；
- 左表过滤条件禁止出现在on子句后面。
```



