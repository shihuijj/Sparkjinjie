# 本地数据集上传到数据仓库Hive

本次实验介绍将实验数据集从本地上传到数据仓库Hive的工作
 
### 所需知识储备
- Linux系统基本命令
- Hadoop项目结构
- 分布式文件系统HDFS概念及其基本原理
- 数据仓库概念及其基本原理
- 数据仓库Hive概念及其基本原理

### 训练技能
- Hadoop的安装与基本操作
- HDFS的基本操作
- 数据仓库Hive的安装与基本操作
- 基本的数据预处理方法

# 实验步骤
- 数据集下载与查看
- 数据集的预处理
- 把数据集导入分布式文件系统HDFS中
- 在数据仓库Hive上创建数据库

### 1. 数据集下载与查看
本案例采用的数据集压缩包为data_format.zip(已经下载到本地的/data/spark/data_format.tar.gz)，该数据集压缩包是淘宝2015年双11前6个月(包含双11)的交易数据(交易数据有偏移)，里面包含3个文件，分别是用户行为日志文件user_log.csv 、回头客训练集train.csv 、回头客测试集test.csv. 下面列出这3个文件的数据格式定义：

**用户行为日志user_log.csv**，日志中的字段定义如下：

| 字段         | 含义  |
| ------------| ------------ |
| user_id     | 买家id  |
| item_id     | 商品id  |
| cat_id      | 商品类别id  |
| merchant_id | 卖家id  |
| brand_id    | 品牌id  |
| month       | 交易时间:月  |
| day         | 交易事件:日  |
| action      | 行为,取值范围{0,1,2,3},0表示点击，1表示加入购物车，2表示购买，3表示关注商品 |
| age_range   | 买家年龄分段：1表示年龄<18,2表示年龄在[18,24]，3表示年龄在[25,29]，4表示年龄在[30,34]，5表示年龄在[35,39]，6表示年龄在[40,49]，7和8表示年龄>=50,0和NULL则表示未知 |
| gender      | 性别:0表示女性，1表示男性，2和NULL表示未知  |
| province    | 收货地址省份  |


**回头客训练集train.csv** 和 **回头客测试集test.csv**，训练集和测试集拥有相同的字段，字段定义如下：

| 字段         | 含义  |
| ------------ | ------------ |
| user_id     | 买家id  |
| age_range   | 买家年龄分段：1表示年龄<18，表示年龄在[18,24]，3表示年龄在[25,29]，4表示年龄在[30,34]，5表示年龄在[35,39]，6表示年龄在[40,49]，7和8表示年龄>=50，0和NULL则表示未知  |
| gender      | 性别:0表示女性，1表示男性，2和NULL表示未知  |
| merchant_id | 商家id  |
| label       | 是否是回头客，0值表示不是回头客，1值表示回头客，-1值表示该用户已经超出我们所需要考虑的预测范围。NULL值只存在测试集，在测试集中表示需要预测的值。  |

创建文件夹 /usr/local/dbtaobao，将数据集解压到 /usr/local/dbtaobao 文件夹中,并重命名为 dataset:
```
# mkdir /usr/local/dbtaobao/
# tar zxvf /data/spark/data_format.tar.gz -C /usr/local/dbtaobao
# mv /usr/local/dbtaobao/data_format /usr/local/dbtaobao/dataset
# cd /usr/local/dbtaobao/dataset
# ls
```
查看前五行数据
```
# head -5 user_log.csv
```
结果如下:

> user_id,item_id,cat_id,merchant_id,brand_id,month,day,action,age_range,gender,province
328862,323294,833,2882,2661,08,29,0,0,1,内蒙古
328862,844400,1271,2882,2661,08,29,0,1,1,山西
328862,575153,1271,2882,2661,08,29,0,2,1,山西
328862,996875,1271,2882,2661,08,29,0,1,1,内蒙古

### 2. 数据集的预处理
##### 1. 删除文件第一行记录
> 即字段名称user_log.csv的第一行都是字段名称，我们在文件中的数据导入到数据仓库Hive中时，不需要第一行字段名称，因此，这里在做数据预处理时，删除第一行：
```
# sed -i '1d' user_log.csv //1d表示删除第1行，同理，3d表示删除第3行，nd表示删除第n行
# head -5 user_log.csv
```
可以看到第一行的数据已经被删除

> 328862,323294,833,2882,2661,08,29,0,0,1,内蒙古
328862,844400,1271,2882,2661,08,29,0,1,1,山西
328862,575153,1271,2882,2661,08,29,0,2,1,山西
328862,996875,1271,2882,2661,08,29,0,1,1,内蒙古
328862,1086186,1271,1253,1049,08,29,0,0,2,浙江

##### 2. 获取数据集中的前100000条数据
由于数据集中交易数据太大，这里只截取数据集中在双11的前10000条交易数据作为小数据集small_user_log.csv。下面我们建立一个脚本文件完成上面截取任务，注意脚本文件要和数据集user_log.csv在同一目录(/usr/local/dbtaobao/dataset)下:

```

# gedit predeal.sh

```

写入以下代码：

```
#!/bin/bash
#下面设置输入文件，把用户执行predeal.sh命令时提供的第一个参数作为输入文件名称
infile=$1
#下面设置输出文件，把用户执行predeal.sh命令时提供的第二个参数作为输出文件名称
outfile=$2
#注意！！最后的$infile > $outfile必须跟在}’这两个字符的后面
awk -F "," 'BEGIN{
      id=0;
    }
    {
        if($6==11 && $7==11){
            id=id+1;
            print $1","$2","$3","$4","$5","$6","$7","$8","$9","$10","$11
            if(id==10000){
                exit
            }
        }
    }' $infile > $outfile
```

下面就可以执行predeal.sh脚本文件，截取数据集中在双11的前10000条交易数据作为小数据集small_user_log.csv，命令如下：

```
# chmod +x ./predeal.sh
# ./predeal.sh ./user_log.csv ./small_user_log.csv
```

##### 3. 将数据上传到HDFS
下面要把small_user_log.csv中的数据最终导入到数据仓库Hive中。为了完成这个操作，我们会首先把这个文件上传到分布式文件系统HDFS中，然后，在Hive中创建外部表，完成数据的导入。(如果启动Hadoop过程中提示:Are you sure you want to continue connecting (yes/no)? 输入yes即可)
> 启动HDFS(实验环境已安装好HDFS)
```
# service ssh restart
# /usr/local/hadoop/sbin/start-dfs.sh
# /usr/local/hadoop/sbin/start-yarn.sh
```
现在，我们要把Linux本地文件系统中的user_log.csv上传到分布式文件系统HDFS中，存放在HDFS中的“/dbtaobao/dataset”目录下。请执行下面命令，在HDFS的根目录下面创建一个新的目录dbtaobao，并在这个目录下创建一个子目录dataset，如下：
```
# cd /usr/local/hadoop
# ./bin/hadoop fs -mkdir -p /dbtaobao/dataset/user_log
```
然后，把Linux本地文件系统中的small_user_log.csv上传到分布式文件系统HDFS的“/dbtaobao/dataset”目录下，命令如下：
```
# ./bin/hadoop fs -put /usr/local/dbtaobao/dataset/small_user_log.csv /dbtaobao/dataset/user_log
```
下面可以查看一下HDFS中的small_user_log.csv的前10条记录，命令如下：
```
# ./bin/hadoop fs -cat /dbtaobao/dataset/user_log/small_user_log.csv | head -10
```
结果如下:

> 328862,406349,1280,2700,5476,11,11,0,0,1,四川

328862,406349,1280,2700,5476,11,11,0,7,1,重庆市

328862,807126,1181,1963,6109,11,11,0,1,0,上海市

328862,406349,1280,2700,5476,11,11,2,6,0,台湾

328862,406349,1280,2700,5476,11,11,0,6,2,甘肃

328862,406349,1280,2700,5476,11,11,0,4,1,甘肃

328862,406349,1280,2700,5476,11,11,0,5,0,浙江

328862,406349,1280,2700,5476,11,11,0,3,2,澳门

328862,406349,1280,2700,5476,11,11,0,7,1,台湾

234512,399860,962,305,6300,11,11,0,4,1,安徽

##### 4. 配置Mysql
本次实验使用Mysql作为Hive的元数据，需在mysql中进行相关配置。
> 启动MySql
```
# mysqld & //此命令会使mysql阻塞在启动界面，按回车键即可
```

在mysql中创建hive数据库，并赋予hive用户权限
```
# mysql -u root -p
// 提示输入密码，默认密码为 123456
mysql> grant all on *.* to hive@localhost identified by 'hive';
mysql> alter database hive character set latin1;
mysql> flush privileges;
//执行完毕后退出mysql shell
mysql> exit
```

##### 5. 将数据导入Hive仓库
>由于Hive是基于Hadoop的数据仓库，使用HiveQL语言撰写的查询语句，最终都会被Hive自动解析成MapReduce任务由Hadoop去具体执行，因此，需要启动Hadoop，然后再启动Hive。由于前面我们已经启动了Hadoop，所以，这里不需要再次启动Hadoop。下面，在这个新的终端中执行下面命令进入Hive：
```
# cd /usr/local/hive
# ./bin/hive
```
如果出现cannot access ‘/usr/local/spark/lib/spark-assembly-*.jar的错误，vi /usr/local/hive/bin/hive，将sparkAssemblyPath改为:sparkAssemblyPath=ls ${SPARK_HOME}/jars/*.jar 。启动成功以后，就进入了“hive>”命令提示符状态，可以输入类似SQL语句的HiveQL语句。下面，我们要在Hive中创建一个数据库dbtaobao，命令如下：
```
hive>  create database dbtaobao;
hive>  use dbtaobao;
```
创建外部表
关于数据仓库Hive的内部表和外部表的区别，请读者自行了解本教程采用外部表方式。这里我们要分别在数据库dbtaobao中创建一个外部表user_log，它包含字段（user_id,item_id,cat_id,merchant_id,brand_id,month,day,action,age_range,gender,province）,请在hive命令提示符下输入如下命令：
```
hive>  CREATE EXTERNAL TABLE dbtaobao.user_log(user_id INT,item_id INT,cat_id INT,merchant_id INT,brand_id INT,month STRING,day STRING,action INT,age_range INT,gender INT,province STRING) COMMENT 'Welcome to xmu dblab,Now create dbtaobao.user_log!' ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE LOCATION '/dbtaobao/dataset/user_log';
```
查询数据
上面已经成功把HDFS中的“/dbtaobao/dataset/user_log”目录下的small_user_log.csv数据加载到了数据仓库Hive中，我们现在可以使用下面命令查询一下：
```
hive>  select * from user_log limit 10;
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

操作成功后,退出Hive
```
hive>  exit;
```
