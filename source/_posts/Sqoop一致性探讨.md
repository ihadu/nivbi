---
title: Sqoop一致性探讨
date: 2021-10-25 13:49:32
tags: sqoop
---
## Sqoop导入导出Null存储一致性问题
Hive中的Null在底层是以“\N”来存储，而MySQL中的Null在底层就是Null，为了保证数据两端的一致性。在导出数据时采用--input-null-string和--input-null-non-string两个参数。导入数据时采用--null-string和--null-non-string。
## Sqoop数据导出一致性问题
### 场景1：
如Sqoop在导出到Mysql时，使用4个Map任务，过程中有2个任务失败，那此时MySQL中存储了另外两个Map任务导入的数据，此时老板正好看到了这个报表数据。而开发工程师发现任务失败后，会调试问题并最终将全部数据正确的导入MySQL，那后面老板再次看报表数据，发现本次看到的数据与之前的不一致，这在生产环境是不允许的。

#### 解决方案：
由于Sqoop将导出过程分解为多个事务，因此失败的导出作业可能会导致部分数据提交到数据库。在某些情况下，这可能进一步导致后续作业因插入冲突而失败，在其他情况下，这又可能导致数据重复。您可以通过--staging-table选项指定暂存表来解决此问题，该选项用作用于暂存导出数据的辅助表。最后，分阶段处理的数据将在单个事务中移至目标表
##### 命令：
```shell
sqoop export \
--connect jdbc:mysql://192.168.137.10:3306/user_behavior \
--username root \
--password 123456 \
--table app_cource_study_report \
--columns watch_video_cnt,complete_video_cnt,dt \
--fields-terminated-by "\t" \
--export-dir "/user/hive/warehouse/tmp.db/app_cource_study_analysis_${day}" \
--staging-table app_cource_study_report_tmp \
--clear-staging-table \
--input-null-string '\N'
```

