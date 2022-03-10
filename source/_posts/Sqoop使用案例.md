---
title: Sqoop使用案例
date: 2021-10-25 13:47:30
tags: sqoop
---
## Sqoop 原理
将导入或导出命令翻译成 mapreduce 程序来实现。
在翻译出的 mapreduce 中主要是对 inputformat 和 outputformat 进行定制。
### 测试 Sqoop 是否能够成功连接数据库
```shell
$ bin/sqoop list-databases --connect jdbc:mysql://hadoop102:3306/
--username root --password 000000
```
## Sqoop 的简单使用案例
###  导入数据
在 Sqoop 中，“导入”概念指：从非大数据集群（RDBMS）向大数据集群（HDFS，HIVE，
HBASE）中传输数据，叫做：导入，即使用 import 关键字。
#### （1）全部导入
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t"
```
#### （2）查询导入
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--query 'select name,sex from staff where id <=1 and $CONDITIONS;'
```
提示：`must contain '$CONDITIONS' in WHERE clause`.
如果 query 后使用的是双引号，则$CONDITIONS 前必须加转移符，防止 shell 识别为自己的
变量。
#### （3）导入指定列
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--columns id,sex \
--table staff
```
提示：columns 中如果涉及到多列，用逗号分隔，分隔时不要添加空格
#### （4）使用 sqoop 关键字筛选查询导入数据
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--table staff \
--where "id=1"
```
#### RDBMS 到 Hive
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--hive-import \
--fields-terminated-by "\t" \
--hive-overwrite \
--hive-table staff_hive
```
提示：该过程分为两步，第一步将数据导入到 HDFS，第二步将导入到 HDFS 的数据迁移到
Hive 仓库，第一步默认的临时目录是/user/atguigu/表名
#### RDBMS 到 Hbase
```shell
$ bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table company \
--columns "id,name,sex" \
--column-family "info" \
--hbase-create-table \
--hbase-row-key "id" \
--hbase-table "hbase_company" \
--num-mappers 1 \
--split-by id
```
提示：sqoop1.4.6 只支持 HBase1.0.1 之前的版本的自动创建 HBase 表的功能
解决方案：手动创建 HBase 表

### 导出数据
在 Sqoop 中，“导出”概念指：从大数据集群（HDFS，HIVE，HBASE）向非大数据集群
（RDBMS）中传输数据，叫做：导出，即使用 export 关键字。
#### HIVE/HDFS 到 RDBMS
```shell
$ bin/sqoop export \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--export-dir /user/hive/warehouse/staff_hive \
--input-fields-terminated-by "\t"
```
提示：Mysql 中如果表不存在，不会自动创建
### 脚本打包
使用 opt 格式的文件打包 sqoop 命令，然后执行
#### 1) 创建一个.opt 文件
```shell
$ mkdir opt
$ touch opt/job_HDFS2RDBMS.opt
```
#### 2) 编写 sqoop 脚本
```shell
$ vi opt/job_HDFS2RDBMS.opt
export
--connect
jdbc:mysql://hadoop102:3306/company
--username
root
--password
000000
--table
staff
--num-mappers
1
--export-dir
/user/hive/warehouse/staff_hive
--input-fields-terminated-by
"\t"
```
#### 3) 执行该脚本
```shell
$ bin/sqoop --options-file opt/job_HDFS2RDBMS.opt
```


