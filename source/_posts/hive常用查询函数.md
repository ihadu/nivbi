---
title: Hive常用查询函数
date: 2021-10-15 15:41:17
tags: hive
---
# 常用查询函数
## 1 空字段赋值NVL函数
1.函数说明
&emsp;&emsp;NVL：给值为NULL的数据赋值，它的格式是NVL( string1, replace_with)。它的功能是如果string1为NULL，则NVL函数返回replace_with的值，否则返回string1的值，如果两个参数都为NULL ，则返回NULL。

2.数据准备：采用员工表

3.查询：如果员工的comm为NULL，则用-1代替
```shell
hive (default)> select nvl(comm,-1) from emp;
```
4.查询：如果员工的comm为NULL，则用领导id代替
```shell
hive (default)> select nvl(comm,mgr) from emp;
```

## 2.CASE WHEN函数
1. 数据准备

| name  | dept_id  | sex  |
| :------------: | :------------: | :------------: |
| 悟空  | A  | 男  |
| 大海  | A  | 男  |
| 宋宋  | B  | 男  |
| 凤姐  | A  | 女  |
| 婷姐  | B  | 女  |
| 婷婷  | B  | 女 |

2．需求
求出不同部门男女各多少人。结果如下：
```
A     2       1
B     1       2
```

3．创建本地emp_sex.txt，导入数据
[ihadu@hadoop102 datas]$ vi emp_sex.txt
悟空	A	男
大海	A	男
宋宋	B	男
凤姐	A	女
婷姐	B	女
婷婷	B	女

4．创建hive表并导入数据
```sql
create table emp_sex(
name string,
dept_id string,
sex string)
row format delimited fields terminated by "\t";

load data local inpath '/opt/module/datas/emp_sex.txt' into table emp_sex;
```
5．按需求查询数据
```sql
select
  dept_id,
  sum(case sex when '男' then 1 else 0 end) male_count,
  sum(case sex when '女' then 1 else 0 end) female_count
from
  emp_sex
group by
  dept_id;
```

## 3.行转列
1．相关函数说明
&emsp;&emsp;CONCAT(string A/col, string B/col…)：返回输入字符串连接后的结果，支持任意个输入字符串;
&emsp;&emsp;CONCAT_WS(separator, str1, str2,...)：它是一个特殊形式的 CONCAT()。第一个参数剩余参数间的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将为 NULL。这个函数会跳过分隔符参数后的任何 NULL 和空字符串。分隔符将被加到被连接的字符串之间;
&emsp;&emsp;COLLECT_SET(col)：函数只接受基本数据类型，它的主要作用是将某字段的值进行去重汇总，产生array类型字段。

2．数据准备

| name  | constellation  | blood_type  |
| :------------: | :------------: | :------------: |
| 孙悟空  | 白羊座  | A  |
| 大海  | 射手座  | A  |
| 宋宋  | 白羊座  | B  |
| 猪八戒  | 白羊座  | A  |
| 凤姐  | 射手座  | A  |

3．需求
把星座和血型一样的人归类到一起。结果如下：
```
射手座,A            大海|凤姐
白羊座,A            孙悟空|猪八戒
白羊座,B            宋宋
```
4．创建本地constellation.txt，导入数据
```shell
[ihadu@hadoop102 datas]$ vi person_info.txt
孙悟空	白羊座	A
大海	 射手座	A
宋宋	 白羊座	B
猪八戒   白羊座	A
凤姐	  射手座	A
```
5．创建hive表并导入数据
```sql
create table person_info(
name string,
constellation string,
blood_type string)
row format delimited fields terminated by "\t";

load data local inpath “/opt/module/datas/person_info.txt” into table person_info;
```
6．按需求查询数据
```sql
select
    t1.base,
    concat_ws('|', collect_set(t1.name)) name
from
    (select
        name,
        concat(constellation, ",", blood_type) base
    from
        person_info) t1
group by
    t1.base;
```

## 4.列转行
1．函数说明
Explode(col)：将hive一列中复杂的array或者map结构拆分成多行。

Lateral view
用法：LATERAL VIEW udtf(expression) tableAlias AS columnAlias
解释：用于和split, explode等UDTF一起使用，它能够将一列数据拆成多行数据，在此基础上可以对拆分后的数据进行聚合。

2．数据准备

| movie  | category |
| :------------: | :------------: |
| 《金刚川》| 悬疑,动作,科幻,剧情  |
| 《我和我的家乡》  | 悬疑,警匪,动作,心理,剧情  |
| 《心灵奇旅》  | 战争,动作,灾难  |

3．需求
将电影分类中的数组数据展开。结果如下：
```shell
《金刚川》      悬疑
《金刚川》      动作
《金刚川》      科幻
《金刚川》      剧情
《我和我的家乡》   悬疑
《我和我的家乡》   警匪
《我和我的家乡》   动作
《我和我的家乡》   心理
《我和我的家乡》   剧情
《心灵奇旅》      战争
《心灵奇旅》      动作
《心灵奇旅》      灾难
```
4．创建本地movie.txt，导入数据
```shell
[ihadu@hadoop102 datas]$ vi movie.txt
《疑犯追踪》	悬疑,动作,科幻,剧情
《Lie to me》	悬疑,警匪,动作,心理,剧情
《战狼2》	战争,动作,灾难
```
5．创建hive表并导入数据
```sql
create table movie_info(
    movie string,
    category array<string>)
row format delimited fields terminated by "\t"
collection items terminated by ",";

load data local inpath "/opt/module/datas/movie.txt" into table movie_info;
```
6．按需求查询数据
```sql
select
    movie,
    category_name
from
    movie_info lateral view explode(category) table_tmp as category_name;
```

## 5.窗口函数
1．相关函数说明
OVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化
CURRENT ROW：当前行
n PRECEDING：往前n行数据
n FOLLOWING：往后n行数据
UNBOUNDED：起点，UNBOUNDED PRECEDING 表示从前面的起点， UNBOUNDED FOLLOWING表示到后面的终点
LAG(col,n)：往前第n行数据
LEAD(col,n)：往后第n行数据
NTILE(n)：把有序分区中的行分发到指定数据的组中，各个组有编号，编号从1开始，对于每一行，NTILE返回此行所属的组的编号。注意：n必须为int类型。

2．数据准备：name，orderdate，cost
```
jack,2017-01-01,10
tony,2017-01-02,15
jack,2017-02-03,23
tony,2017-01-04,29
jack,2017-01-05,46
jack,2017-04-06,42
tony,2017-01-07,50
jack,2017-01-08,55
mart,2017-04-08,62
mart,2017-04-09,68
neil,2017-05-10,12
mart,2017-04-11,75
neil,2017-06-12,80
mart,2017-04-13,94
```

3．需求
（1）查询在2017年4月份购买过的顾客及总人数
（2）查询顾客的购买明细及月购买总额
（3）上述的场景,要将cost按照日期进行累加
（4）查询顾客上次的购买时间
（5）查询前20%时间的订单信息

4．创建本地business.txt，导入数据
```
[ihadu@hadoop102 datas]$ vi business.txt
```

5．创建hive表并导入数据
```sql
create table business(
name string,
orderdate string,
cost int
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

load data local inpath "/opt/module/datas/business.txt" into table business;
```

6．按需求查询数据
（1）查询在2017年4月份购买过的顾客及总人数
```sql
select name,count(*) over ()
from business
where substring(orderdate,1,7) = '2017-04'
group by name;
```

（2）查询顾客的购买明细及月购买总额
```sql
select
	name,
	orderdate,
	cost,
	sum(cost) over(partition by month(orderdate))
from business;
```

（3）上述的场景,要将cost按照日期进行累加
```sql
select name,orderdate,cost,
sum(cost) over() as sample1,--所有行相加
sum(cost) over(partition by name) as sample2,--按name分组，组内数据相加
sum(cost) over(partition by name order by orderdate) as sample3,--按name分组，组内数据累加
sum(cost) over(partition by name order by orderdate rows between UNBOUNDED PRECEDING and current row ) as sample4 ,--和sample3一样,由起点到当前行的聚合
sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and current row) as sample5, --当前行和前面一行做聚合
sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING AND 1 FOLLOWING ) as sample6,--当前行和前边一行及后面一行
sum(cost) over(partition by name order by orderdate rows between current row and UNBOUNDED FOLLOWING ) as sample7 --当前行及后面所有行
from business;
```

（4）查看顾客上次的购买时间
```sql
select name,orderdate,cost,
lag(orderdate,1,'1900-01-01') over(partition by name order by orderdate ) as time1,
lag(orderdate,2) over (partition by name order by orderdate) as time2
from business;
```

（5）查询前20%时间的订单信息
```sql
select * from (
    select name,orderdate,cost, ntile(5) over(order by orderdate) sorted
    from business
) t
where sorted = 1;
```

6.Rank函数
1．函数说明
- RANK() 排序相同时会重复，总数不会变
- DENSE_RANK() 排序相同时会重复，总数会减少
- ROW_NUMBER() 会根据顺序计算

2．数据准备

| name  | subject  | score  |
| :------------: | :------------: | :------------: |
| 孙悟空  | 语文  | 87  |
| 孙悟空  | 数学  | 95  |
| 孙悟空  | 英语  | 68  |
| 大海  | 语文  | 94  |
| 大海  | 数学  | 56  |
| 大海  | 英语  | 84  |
| 宋宋  | 语文  | 64  |
| 宋宋  | 数学  | 86  |
| 宋宋  | 英语  | 84  |
| 婷婷  | 语文  | 65  |
| 婷婷  | 数学  | 85  |
| 婷婷  | 英语  | 78  |

3．需求
计算每门学科成绩排名。

4．创建本地movie.txt，导入数据
```shell
[ihadu@hadoop102 datas]$ vi score.txt
```

5．创建hive表并导入数据
```sql
create table score(
name string,
subject string, 
score int) 
row format delimited fields terminated by "\t";
load data local inpath '/opt/module/datas/score.txt' into table score;
```

6．按需求查询数据
```sql
select name,
subject,
score,
rank() over(partition by subject order by score desc) rp,
dense_rank() over(partition by subject order by score desc) drp,
row_number() over(partition by subject order by score desc) rmp
from score;
```
查询结果：
```shell
name    subject score   rp      drp     rmp
孙悟空  数学    95      1       1       1
宋宋    数学    86      2       2       2
婷婷    数学    85      3       3       3
大海    数学    56      4       4       4
宋宋    英语    84      1       1       1
大海    英语    84      1       1       2
婷婷    英语    78      3       2       3
孙悟空  英语    68      4       3       4
大海    语文    94      1       1       1
孙悟空  语文    87      2       2       2
婷婷    语文    65      3       3       3
宋宋    语文    64      4       4       4
````


