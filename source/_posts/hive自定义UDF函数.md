---
title: Hive自定义UDF函数
date: 2021-10-13 16:55:03
tags: hive
---

### 1. Hive自定义函数介绍
当Hive提供的内置函数无法满足你的业务处理需要时，此时可以考虑使用用户自定义函数（UDF: user-defined function）。
Hive 中常用的UDF有如下三种:
- UDF
  一条记录使用函数后输出还是一条记录，比如:upper/substr;
- UDAF(User-Defined Aggregation Funcation)
  多条记录使用函数后输出还是一条记录，比如: count/max/min/sum/avg;
- UDTF(User-Defined Table-Generating Functions)
  一条记录使用函数后输出多条记录，比如: lateral view explore();
### 2. Hive自定义函数开发
需求:开发自定义函数，使得在指定字段前加上“Hello:”字样。Hive 中 UDF函数开发步骤:
(1）继承UDF 类。
(2）重写evaluate方法，该方法支持重载，每行记录执行一次evaluate方法。

#### ##### 注意：
1 UDF必须要有返回值,可以是null,但是不能为 void.
2 推荐使用 Text/LongWritable等Hadoop的类型,而不是Java类型(当然使用 Java类型也是可以的)。

功能实现:
##### ( 1)pom.xml中添加UDF函数开发的依赖包。
```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.version>2.6.0-cdh5.7.0</hadoop.version>
    <hive.version>1.1.0-cdh5.7.0</hive.version>
</properties>
<!--CDH 版本建议大家添加一个repository-->
<repositories>
  <repository>
    <id>cloudera</id>
    <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
  </repository>
</repositories>

<dependencies>
<!--Hadoop依赖-->
<dependency>
   <groupId>org.apache.hadoop</groupld>
   <artifactId>hadoop-common</artifactld>
   <version>$ {hadoop.version}</version>
</dependency>

<!--Hive依赖-->
<dependency>
   <groupld>org.apache.hive</groupld>
   <artifactId>hive-exec</artifactId>
   <version>$ { hive.version}</version>
</dependency>

<dependency>
   <groupld>org.apacne.hive</groupId>
   <artifactld>hive-jdbc</artifactId>
   <version>$ { hive.version}</version>
</dependency>
</dependencies>
```

##### (2）开发UDF函数。
```java
package com.kgc.bigdata.hadoop.hive;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
/**
*功能:输入xxx，输出:Hello: xxx
*
*开发UDF 函数的步骤
 
* 1) extends UDF
*2）重写evaluate方法，注意该方法是支持重载的
*/


public class HelloUDF extends UDF{
/**
*对于UDF 函数的evaluate的参数和返回值，个人建议使用Writable* @param name
* @return
*/
public Text evaluate(Text name){
return new Text("Hello: " + name);
    }
public Text evaluate(Text name,IntWritable age){
return new Text("Hello: " +name + " , age :" + age);
    }
/功能测试
public static void main(String[] args){
HelloUDF udf = new HelloUDF(
System.out.println(udf.evaluate(new Text("zhangsan")));
System.out.println(udf.evaluate(new Text("zhangsan"), new IntWritable(20)));
   }
}
```
(3）编译jar包上传到服务器。
(4)将自定义UDF 函数添加到Hive 中去。
```shell
add JAR/home/hadoop/lib/hive-1.0.jar;
create temporary function sayHello as 'com.kgc.bigdata.hadoop.hive.HelloUDF';
```
(5)使用自定义函数。
//通过show functions可以看到我们自定义的sayHello函数show functions;
//将员工表的ename作为自定义UDF函数的参数值，即可查看到最终的输出结果
```sql
select empno, ename, sayHello(ename) from emp;
```

