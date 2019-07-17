# Hive数据分析

本文介绍大数据课程实验案例“淘宝双11数据分析与预测”的第二个步骤，Hive数据分析。在实践本步骤之前，请先完成该实验案例的第一个步骤大数据案例——本地数据集上传到数据仓库Hive。这里假设你已经完成了前面的第一个步骤。

### 所需知识储备
- 数据仓库Hive概念及其基本原理
- SQL语句、数据库查询分析

### 训练技能
- 数据仓库Hive基本操作
- 创建数据库和表
- 使用SQL语句进行查询分析

### 任务清单
- 1.启动Hadoop和Hive
- 2.创建数据库和表
- 3.简单查询分析
- 4.查询条数统计分析
- 5.关键字条件查询分析
- 6.根据用户行为分析
- 7.用户实时查询分析

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

### 2. 操作Hive
在“hive>”命令提示符状态下执行下面命令：

```
hive> use dbtaobao; -- 使用dbtaobao数据库
hive> show tables; -- 显示数据库中所有表。
hive> show create table user_log; -- 查看user_log表的各种属性；
```

执行结果如下：

>OK
CREATE EXTERNAL TABLE `user_log`(
  `user_id` int,
  `item_id` int,
  `cat_id` int,
  `merchant_id` int,
  `brand_id` int,
  `month` string,
  `day` string,
  `action` int,
  `age_range` int,
  `gender` int,
  `province` string)
COMMENT 'Welcome to xmu dblab,Now create dbtaobao.user_log!'
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ','
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://localhost:9000/dbtaobao/dataset/user_log'
TBLPROPERTIES (
  'COLUMN_STATS_ACCURATE'='false',
  'numFiles'='0',
  'numRows'='-1',
  'rawDataSize'='-1',
  'totalSize'='0',
  'transient_lastDdlTime'='1523852194')

可以执行下面命令查看表的简单结构：

```
hive> desc user_log;
```

执行结果如下：

>OK

user_id             	int                 	                    

item_id             	int                 	                    

cat_id              	int                 	                    

merchant_id         	int                 	                    

brand_id            	int                 	                    

month               	string              	                    

day                 	string              	                    

action              	int                 	                    

age_range           	int                 	                    

gender              	int                 	                    

province            	string              	                    

Time taken: 0.11 seconds, Fetched: 11 row(s)

### 3. 简单查询分析
##### 1. 先测试一下简单的指令：

```
hive> select brand_id from user_log limit 10; -- 查看日志前10个交易日志的商品品牌
```

> OK

5476

5476

6109

5476

5476

5476

5476

5476

5476

6300

##### 2. 如果要查出每位用户购买商品时的多种信息，输出语句格式为 select 列1，列2，….，列n from 表名；比如我们现在查询前20个交易日志中购买商品时的时间和商品的种类

```
hive> select month,day,cat_id from user_log limit 20;
```

执行结果如下：
> OK

11	11	1280

11	11	1280

11	11	1181

11	11	1280

11	11	1280

11	11	1280

11	11	1280

11	11	1280

11	11	1280

11	11	962

11	11	81

11	11	1432

11	11	389

11	11	1208

11	11	1611

11	11	420

11	11	1611

11	11	1432

11	11	389

11	11	1432

##### 3. 有时我们在表中查询可以利用嵌套语句，如果列名太复杂可以设置该列的别名，以简化我们操作的难度，以下我们可以举个例子：

```
hive> select ul.at, ul.ci  from (select action as at, cat_id as ci from user_log) as ul limit 20;
```

执行结果如下：
> OK

0   1280

0   1280

0   1181

2   1280

0   1280

0   1280

0   1280

0   1280

0   1280

0   962

2   81

2   1432

0   389

2   1208

0   1611

0   420

0   1611

0   1432

0   389

0   1432

这里简单的做个讲解，action as at ,cat_id as ci就是把action 设置别名 at ,cat_id 设置别名 ci，FROM的括号里的内容我们也设置了别名ul，这样调用时用ul.at,ul.ci,可以简化代码。

### 4. 查询条数统计分析
经过简单的查询后我们同样也可以在select后加入更多的条件对表进行查询,下面可以用函数来查找我们想要的内容。
##### 1. 用聚合函数count()计算出表内有多少条行数据

```
hive> select count(*) from user_log; -- 用聚合函数count()计算出表内有多少条行数据
```

执行结果如下：
> OK
10000

我们可以看到，得出的结果为OK下的那个数字10000

##### 2. 在函数内部加上distinct，查出uid不重复的数据有多少条
下面继续执行操作：

```
hive> select count(distinct user_id) from user_log; -- 在函数内部加上distinct，查出user_id不重复的数据有多少条
```

执行结果如下：

> OK
358

##### 3. 查询不重复的数据有多少条(为了排除客户刷单情况)

```
hive> select count(*) from (select user_id,item_id,cat_id,merchant_id,brand_id,month,day,action from user_log group by user_id,item_id,cat_id,merchant_id,brand_id,month,day,action having count(*)=1)a;
```

执行结果如下：
> OK
4754

可以看出，排除掉重复信息以后，只有4754条记录。
注意：嵌套语句最好取别名，就是上面的a，否则很容易出现如下错误.

> FAILED: ParseException line 1:131 cannot recognize input near '< EOF >' '< EOF >' in subquery source.

### 5．关键字条件查询分析
##### 1. 以关键字的存在区间为条件的查询
使用where可以缩小查询分析的范围和精确度，下面用实例来测试一下。
(1)查询双11那天有多少人购买了商品

```
hive> select count(distinct user_id) from user_log where action='2';
```

执行结果如下：
> OK
358

##### 2. 关键字赋予给定值为条件，对其他数据进行分析
取给定时间和给定品牌，求当天购买的此品牌商品的数量

```
hive> select count(*) from user_log where action='2' and brand_id=2661;
```

> OK
3

### 6．根据用户行为分析
##### 1. 查询一件商品在某天的购买比例或浏览比例

```
hive> select count(distinct user_id) from user_log where action='2'; -- 查询有多少用户在双11购买了商品
```

执行结果如下：
> OK
358

```
hive> select count(distinct user_id) from user_log; -- 查询有多少用户在双11点击了该店
```

执行结果如下：
> OK
358

根据上面语句得到购买数量和点击数量，两个数相除即可得出当天该商品的购买率。

##### 2. 查询双11那天，男女买家购买商品的比例

```
hive> select count(*) from user_log where gender=0; --查询双11那天女性购买商品的数量
```

执行结果如下：
> OK
3361

```
hive> select count(*) from user_log where gender=1; --查询双11那天男性购买商品的数量
```

执行结果如下：
> OK
3299

上面两条语句的结果相除，就得到了要求的比例。

##### 3. 给定购买商品的数量范围，查询某一天在该网站的购买该数量商品的用户id

```
hive> select user_id from user_log where action='2' group by user_id having count(action='2')>5; -- 查询某一天在该网站购买商品超过5次的用户id
```

执行结果最后10条数据如下：
> 356408
366342

370679

378206

379005

389295

396129

407719

409280

4229117

### 7. 用户实时查询分析
不同的品牌的浏览次数

```
hive> create table scan(brand_id INT,scan INT) COMMENT 'This is the search of bigdatataobao' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TEXTFILE; -- 创建新的数据表进行存储
hive> insert overwrite table scan select brand_id,count(action) from user_log where action='2' group by brand_id; --导入数据
hive> select * from scan; -- 显示结果
```

执行结果最后9条数据如下：

8309	1

8319	1

8322	1

8329	3

8357	2

8391	1

8396	3

8410	1

8461	1

到这里，Hive数据分析实验顺利结束。