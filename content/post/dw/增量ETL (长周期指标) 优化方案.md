---
title: "增量ETL (长周期指标) 优化方案"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["ETL", "大数据"]
categories: ["SQL"]
author: "ChavinKing"
---



​	在日常数据处理过程中避免不了要计算跨长周期数据指标统计需求，类似于如下：

​       1、 统计每个城市(过去30天)用户浏览次数；

​           统计每个城市(本年)用户浏览次数；

​           统计每个城市(历史至今)用户浏览次数；

​       2、统计每个城市(过去30天|本年|历史至今)交易用户数；

​       3、数据集部分数据行存在状态变化数据指标需求。

​    通常面对以上数据指标需求，最大的问题是跨长周期数据量往往是巨大的或者数据周期范围不固定。下面依次求解。

场景1：统计(过去30天|本年|历史至今)用户浏览次数？

​    常规解决方案如下：

```sql
select

 city_id as city_id,
 count(1) as pv
 
from events t1
where p_date >= date_add(current_date(),-31) 
and p_date <= date_add(current_date(),-1)
group by city_id
;
```

​    面临的问题，过去30天埋点事件数据量巨大，服务器无法承受且此简单SQL运行时间太长。

解决思路：

​    通过构建增量更新的中间表，降低结果SQL扫描数据量，从而优化任务的执行时间和资源消耗问题。

1、 设计中间表

```sql
create table e_city_day_indi(
    city_id   bigint
    pv     int
)partitioned by(p_date string)
stored as parquet
;
```

​    e_city_day_indi有两种加载方法，我分别称其为日增量更新方法和循环迭代更新方法，用于方便后续问题描述。

​        l 日增量更新：每天汇总当天数据计算出每个城市PV，并且存储到当天日期分区；

​             优点 更通用；

​             缺点 不能完全避免多分区扫描，且可能造成小文件过多。

​        l 循环迭代更新：每天新增数据和前一天快照数据做合并，并且将结果存储到当天日期分区。

​             优点 应用端仅扫描最新分区即可满足需求；

​             缺点 应用面较窄，部分长周期需求无法满足。

2、 ETL实现

| 日增量更新   | insert overwrite table e_city_day_indi partition(p_date)selectcity_id as city_id,count(1) as pv,date_add(current_date(),-1) as p_datefrom eventswhere p_date = date_add(current_date(),-1)group by city_id; |
| ------------ | ------------------------------------------------------------ |
| 循环迭代更新 | insert overwrite table e_city_day_indi partition(p_date)selectcity_id,sum(pv) as pv,date_add(current_date(),-1) as p_datefrom (selectcity_id as city_id,count(1) as pvfrom eventswhere p_date = date_add(current_date(),-1)group by city_idunion allselectcity_id,pvfrom e_city_day_indiwhere p_date = date_add(current_date(),-2)) tgroup by city_id; |

3、 优化后取数逻辑

| 日增量更新取数SQL   | selectcity_id,sum(pv) as pvfrom e_city_day_indiwhere p_date >= date_add(current_date(),-31) and p_date <= date_add(current_date(),-1)group by city_id; |
| ------------------- | ------------------------------------------------------------ |
| 循环迭代更新取数SQL | selectcity_id,pvfrom e_city_day_indiwhere pdate = date_add(current_date(),-1); |

​    以上两种增量加载方法可以满足问题1需求，同时降低了加工数据量。

场景2：统计每个城市(过去30天|本年|历史至今)交易用户数

​     常规解决方案如下：

```sql
select

	city_id,
	count(distinct user_id) as uv
	
from events
where p_date >= date_add(current_date(),-31) 
and p_date <= date_add(current_date(),-1)
group by city_id
;
```

​    典型count distinct类型需求，面临的问题，过去30天埋点事件数据量巨大，服务器无法承受且此简单SQL运行时间太长。

解决思路：

​    通过构建增量更新的中间表，降低结果SQL扫描数据量，从而优化任务的执行时间和资源消耗问题。

1、 设计中间表

```sql
create table e_city_day_indi(
    city_id   bigint,
    user_id   bigint,
    pv     int
) partitioned by(p_date)
stored as parquet
;
```

​    e_city_day_indi数据加载方法在这里仅介绍日增量更新方法。

2、ETL实现

| 日增量更新 | insert overwrite table e_city_day_indi partition(p_date)selectcity_id,user_id,count(1) as pv,date_add(current_date(),-1) as p_datefrom eventswhere p_date = date_add(current_date(),-1)group by city_id,user_id; |
| ---------- | ------------------------------------------------------------ |
|            |                                                              |

3、优化后取数逻辑

| 日增量更新 | selectcity_id,count(distinct user_id) as uv,sum(pv) as pvfrom e_city_day_indiwhere p_date >= date_add(current_date(),-31) and p_date <= date_add(current_date(),-1)group by city_id; |
| ---------- | ------------------------------------------------------------ |
|            |                                                              |

场景3：数据集部分数据行存在状态变化数据指标需求

​    这种需求场景比较特殊，是针对全量数据做update操作，一般性问题是全表数据量较大，但是需要update数据量较小，且基于hive解决方案，update是较难以处理的场景。

​    解决思路：将变化数据和明确不变数据做分离（能否合理做数据热度分离是关键）。具体实施细节是设计分区表，将少量变化数据存储一个分区，将大量不变数据存储在另一个分区，设计历史数据更新的仅扫描活跃分区即可。

​    实施方案：

l 设计目标表两个分区（或者日期分区下设计二级分区）

​           活跃分区：用来存储生命周期未结束且可能存在update操作的数据。

​           非活跃分区：用来存储明确生命周期结束数据。

l 设计日期分区

​           历史日期分区（p_date <= current_date()）存储生命周期结束的数据行；

​           未来日期分区（p_date = ‘9999-12-31’）存储生命周期尚未结束的数据行。

​    以上3种典型场景是做数据开发过程中较基础也是比较常见的增量ETL开发场景，使用合适的方法处理问题可以大大减少资源消耗同时可以有效降低执行时间。