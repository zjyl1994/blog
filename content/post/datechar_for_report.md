---
title: "几种报表SQL常用的时间分组写法"
date: 2019-06-22T13:50:00+08:00
---

> 写了小一个月报表相关的SQL，把最容易忘的记下来

最近在公司的任务多数是写各种报表，SQL最长的格式化完了有30行，各种 JOIN 和 UNION ALL 拼 N 张表里的数据。其中最常用到的就是按照时间分组。

假设有如下表

|员工名(VARCHAR)|交易金额(INT)|时间(DATETIME)|
|----|-----|----|
| 张三 | 66 | 2017-06-23 19:34:00 |
| 张三 | 166 | 2017-06-23 19:36:00 |
| 张三 | 566 | 2017-06-23 20:34:00 |
| 张三 | 934 | 2017-06-24 21:34:00 |
| 李四 | 661 | 2017-07-01 10:36:00 |
| 李四 | 83 | 2017-06-25 19:34:00 |
| 李四 | 23 | 2017-06-26 19:34:00 |

如果想查某个时间段里每个人的总交易金额报表，大概的 SQL 写完应该是这样的

```sql
SELECT `staff_name`,SUM(`money`),(DATE_FORMAT(`time`, '%Y-%m-%d')) AS date_char
FROM table_name
WHERE `time` >= "开始时间" AND `time` <= "结束时间"
GROUP BY staff_name,date_char
```

其中最重要的部分就是 date_char ,它决定了时间分组的粒度。

<!--more-->

常用的时间分组SQL见下表（使用时须将fieldName替换成真实数据库里的字段）

| 粒度 | SQL | date_char输出格式 |
|---|---|---|
|半小时|`(DATE_FORMAT(fieldName - INTERVAL MINUTE(fieldName)%30 MINUTE, '%Y-%m-%d %H:%i:00'))`|2019-06-22 13:30:00|
|小时|`(DATE_FORMAT(fieldName, '%Y-%m-%d %H:00:00'))`|2019-06-22 13:00:00|
|天|`(DATE_FORMAT(fieldName, '%Y-%m-%d'))`|2019-06-22|
|周|`(DATE_FORMAT(fieldName, '%Y(%u)'))`|2019(25)|
|月|`(DATE_FORMAT(fieldName, '%Y-%m'))`|2019-06|
|年|`(DATE_FORMAT(fieldName, '%Y'))`|2019|

使用的时候直接把上面的 SQL 复制出来，修改字段名，然后加上 AS date_char 就好了。

我司早期的报表查询都是用 DATE_FORMAT 把时间转成字符串然后再切割指定长度的（比如天就切10位，月就切7位，年切4位这样的），但是字符串运算永远比直接算数慢，所以说还是我这种方法比较靠谱。

另外要注意，做这种数据报表的时候一定要用 WHERE 提前圈定数据范围，缩小进入 GROUP BY 运算的数据量，要不然一个SQL得跑好久。