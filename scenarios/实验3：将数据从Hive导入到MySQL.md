# 将数据从Hive导入到MySQL
本教程介绍大数据课程实验案例“淘宝双11数据分析与预测”的第三个步骤，将数据从Hive导入到MySQL。从数据导入到MySQL是为了后续数据可视化，服务端读取MySQL中的数据，渲染到前端ECharts页面。

### 所需知识储备
- 数据仓库Hive概念与基本原理
- 关系数据库概念与基本原理
- SQL语句

### 训练技能
- 数据仓库Hive的基本操作
- 关系数据库MySQL的基本操作
- Sqoop工具的使用方法

### 任务清单
- Hive预操作
- 使用Sqoop将数据从Hive导入MySQL

# 实验步骤

### 1. 环境准备

因为需要借助于MySQL保存Hive的元数据，所以如果没有启动MySQL数据库，请首先启动MySQL数据库，请在终端中输入下面命令：

```
# mysqld &
```

由于Hive是基于Hadoop的数据仓库，使用HiveQL语言撰写的查询语句，最终都会被Hive自动解析成MapReduce任务由Hadoop去具体执行，因此，需要启动Hadoop，然后再启动Hive。如果没启动Hadoop,请执行下面命令启动Hadoop：

```
# service ssh restart
# /usr/local/hadoop/sbin/start-dfs.sh
# /usr/local/hadoop/sbin/start-yarn.sh
```

下面，继续执行下面命令启动进入Hive：

```
# cd /usr/local/hive
# ./bin/hive
```

通过上述过程，我们就完成了MySQL、Hadoop和Hive三者的启动。启动成功以后，就进入了“hive>”命令提示符状态，可以输入类似SQL语句的HiveQL语句。

### 2. Hive预操作
##### 1. 创建临时表inner_user_log和inner_user_info

```
hive> create table dbtaobao.inner_user_log(user_id INT,item_id INT,cat_id INT,merchant_id INT,brand_id INT,month STRING,day STRING,action INT,age_range INT,gender INT,province STRING) COMMENT 'Welcome to XMU dblab! Now create inner table inner_user_log ' ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;
```

这个命令执行完以后，Hive会自动在HDFS文件系统中创建对应的数据文件“/user/hive/warehouse/dbtaobao.db/inner_user_log”。

##### 2、将user_log表中的数据插入到inner_user_log,
在[实验1:本地数据集上传到数据仓库Hive]中，我们已经在Hive中的dbtaobao数据库中创建了一个外部表user_log。下面把dbtaobao.user_log数据插入到dbtaobao.inner_user_log表中，命令如下：

```
hive> INSERT OVERWRITE TABLE dbtaobao.inner_user_log select * from dbtaobao.user_log;
```

请执行下面命令查询上面的插入命令是否成功执行：

```
hive> select * from dbtaobao.inner_user_log limit 10;
```

结果如下:
> OK

328862	406349	1280	2700	5476	11	11	0	0	1	四川

328862	406349	1280	2700	5476	11	11	0	7	1	重庆市

328862	807126	1181	1963	6109	11	11	0	1	0	上海市

328862	406349	1280	2700	5476	11	11	2	6	0	台湾

328862	406349	1280	2700	5476	11	11	0	6	2	甘肃

328862	406349	1280	2700	5476	11	11	0	4	1	甘肃

328862	406349	1280	2700	5476	11	11	0	5	0	浙江

328862	406349	1280	2700	5476	11	11	0	3	2	澳门

328862	406349	1280	2700	5476	11	11	0	7	1	台湾

234512	399860	962	305	6300	11	11	0	4	1	安徽


### 3. 使用Sqoop将数据从Hive导入MySQL

##### 1. 登录 MySQL
新建一个终端，执行下面命令：

```
# mysql –u root –p
```
回车后会要求输入Mysql密码,默认密码为 123456

为了简化操作，本教程直接使用root用户登录MySQL数据库，但是，在实际应用中，建议在MySQL中再另外创建一个用户。执行上面命令以后，就进入了“mysql>”命令提示符状态。

##### 2. 创建数据库
```
mysql> show databases; #显示所有数据库
mysql> create database dbtaobao; #创建dbtaobao数据库
mysql> use dbtaobao; #使用数据库
```

##### 3. 创建表
下面在MySQL的数据库dbtaobao中创建一个新表user_log，并设置其编码为utf-8：

```
mysql> CREATE TABLE `dbtaobao`.`user_log` (`user_id` varchar(20),`item_id` varchar(20),`cat_id` varchar(20),`merchant_id` varchar(20),`brand_id` varchar(20), `month` varchar(6),`day` varchar(6),`action` varchar(6),`age_range` varchar(6),`gender` varchar(6),`province` varchar(10)) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

提示：语句中的引号是反引号，不是单引号。需要注意的是，sqoop抓数据的时候会把类型转为string类型，所以mysql设计字段的时候，设置为varchar。

##### 4. 导入数据
```
# cd /usr/local/sqoop
# bin/sqoop export --connect "jdbc:mysql://localhost:3306/dbtaobao?useUnicode=true&characterEncoding=utf-8" --username root --password 123456 --table user_log --export-dir '/user/hive/warehouse/dbtaobao.db/inner_user_log' --fields-terminated-by ',';
```

字段解释 :
> ./bin/sqoop export ##表示数据从 hive 复制到 mysql 中

–connect jdbc:mysql://localhost:3306/dbtaobao

–username root #mysql登陆用户名

–password root #登录密码

–table user_log #mysql 中的表，即将被导入的表名称

–export-dir ‘/user/hive/warehouse/dbtaobao.db/user_log ‘ 
#hive 中被导出的文件

–fields-terminated-by ‘,’ #Hive 中被导出的文件字段的分隔符

结果如下:

> INFO mapreduce.Job: Job job_1523849799701_0014 completed successfully

INFO mapreduce.Job: Counters: 30

File System Counters

FILE: Number of bytes read=0

FILE: Number of bytes written=557120

FILE: Number of read operations=0

FILE: Number of large read operations=0

FILE: Number of write operations=0

HDFS: Number of bytes read=490548

HDFS: Number of bytes written=0

HDFS: Number of read operations=19

HDFS: Number of large read operations=0

HDFS: Number of write operations=0

Job Counters

Launched map tasks=4

Data-local map tasks=4

Total time spent by all maps in occupied slots (ms)=45200

Total time spent by all reduces in occupied slots (ms)=0

Total time spent by all map tasks (ms)=45200

Total vcore-milliseconds taken by all map tasks=45200

Total megabyte-milliseconds taken by all map tasks=46284800

Map-Reduce Framework

Map input records=10000

Map output records=10000

Input split bytes=726

Spilled Records=0

Failed Shuffles=0

Merged Map outputs=0

GC time elapsed (ms)=839

CPU time spent (ms)=7560

Physical memory (bytes) snapshot=684490752

Virtual memory (bytes) snapshot=7840956416

Total committed heap usage (bytes)=586678272

File Input Format Counters Bytes Read=0

File Output Format Counters Bytes Written=0

INFO mapreduce.ExportJobBase: Transferred 479.0508 KB in 27.3117 seconds (17.5401 KB/sec)

INFO mapreduce.ExportJobBase: Exported 10000 records.

##### 4. 查看MySQL中user_log或user_info表中的数据
进入mysql
```
# mysql -u root -p
```

然后执行下面命令查询user_log表中的数据：
```
mysql> use dbtaobao;
mysql> select * from user_log limit 10;
```
(在终端查看输出结果时可能会出现中文乱码,不会影响实验)

从Hive导入数据到MySQL中，成功！