# 利用Spark预测回头客
本教程介绍大数据课程实验案例“淘宝双11数据分析与预测”的第五个步骤，利用Spark预测回头客。


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

通过上述过程，我们就完成了MySQL和Hadoop的启动.


### 2. 预处理test.csv和train.csv数据集
这里列出test.csv和train.csv中字段的描述，字段定义如下：

1. user_id | 买家id
2. age_range | 买家年龄分段：1表示年龄<18,2表示年龄在[18,24]，3表示年龄在[25,29]，4表示年龄在[30,34]，5表示年龄在[35,39]，6表示年龄在[40,49]，7和8表示年龄>=50,0和NULL则表示未知
3. gender | 性别:0表示女性，1表示男性，2和NULL表示未知
4. merchant_id | 商家id
5. label | 是否是回头客，0值表示不是回头客，1值表示回头客，-1值表示该用户已经超出我们所需要考虑的预测范围。NULL值只存在测试集，在测试集中表示需要预测的值。

这里需要预先处理test.csv数据集，把这test.csv数据集里label字段表示-1值剔除掉,保留需要预测的数据.并假设需要预测的数据中label字段均为1.

```
# cd /usr/local/dbtaobao/dataset
# gedit predeal_test.sh
```

上面使用gedit编辑器新建了一个predeal_test.sh脚本文件，请在这个脚本文件中加入下面代码：

```
#!/bin/bash
#下面设置输入文件，把用户执行predeal_test.sh命令时提供的第一个参数作为输入文件名称
infile=$1
#下面设置输出文件，把用户执行predeal_test.sh命令时提供的第二个参数作为输出文件名称
outfile=$2
#注意！！最后的$infile > $outfile必须跟在}’这两个字符的后面
awk -F "," 'BEGIN{
      id=0;
    }
    {
        if($1 && $2 && $3 && $4 && !$5){
            id=id+1;
            print $1","$2","$3","$4","1
            if(id==10000){
                exit
            }
        }
    }' $infile > $outfile
```

下面就可以执行predeal_test.sh脚本文件，截取测试数据集需要预测的数据到test_after.csv，命令如下：
```
# chmod +x ./predeal_test.sh
# ./predeal_test.sh ./test.csv ./test_after.csv
```
train.csv的第一行都是字段名称，不需要第一行字段名称,这里在对train.csv做数据预处理时，删除第一行
```
# sed -i '1d' train.csv
```

然后剔除掉train.csv中字段值部分字段值为空的数据。

```
# cd /usr/local/dbtaobao/dataset
# gedit predeal_train.sh
```

上面使用gedit编辑器新建了一个predeal_train.sh脚本文件，请在这个脚本文件中加入下面代码：

```
#!/bin/bash
#下面设置输入文件，把用户执行predeal_train.sh命令时提供的第一个参数作为输入文件名称
infile=$1
#下面设置输出文件，把用户执行predeal_train.sh命令时提供的第二个参数作为输出文件名称
outfile=$2
#注意！！最后的$infile > $outfile必须跟在}’这两个字符的后面
awk -F "," 'BEGIN{
         id=0;
    }
    {
        if($1 && $2 && $3 && $4 && ($5!=-1)){
            id=id+1;
            print $1","$2","$3","$4","$5
            if(id==10000){
                exit
            }
        }
    }' $infile > $outfile
```

下面就可以执行predeal_train.sh脚本文件，截取测试数据集需要预测的数据到train_after.csv，命令如下：

```
# chmod +x ./predeal_train.sh
# ./predeal_train.sh ./train.csv ./train_after.csv
```

### 预测回头客
1. 创建数据表 rebuy
```
# mysql -u root -p
mysql> use dbtaobao;
mysql> create table rebuy (score varchar(40),label varchar(40));
mysql> exit;
```

2. 将两个数据集分别存取到HDFS中
```
# cd /usr/local/hadoop/
# bin/hadoop fs -mkdir -p /dbtaobao/dataset
# bin/hadoop fs -put /usr/local/dbtaobao/dataset/train_after.csv /dbtaobao/dataset
# bin/hadoop fs -put /usr/local/dbtaobao/dataset/test_after.csv /dbtaobao/dataset
```
3. 启动spark shell
Spark支持通过JDBC方式连接到其他数据库获取数据生成DataFrame。
执行如下命令：
```
# cd /usr/local/spark
# ./bin/spark-shell --jars /usr/local/spark/jars/mysql-connector-java-5.1.46-bin.jar --driver-class-path /usr/local/spark/jars/mysql-connector-java-5.1.46-bin.jar
```

##### 支持向量机SVM分类器预测回头客
这里使用Spark MLlib自带的支持向量机SVM分类器进行预测回头客。在spark-shell中执行如下操作：
1. 导入需要的包
```
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.linalg.{Vectors,Vector}
import org.apache.spark.mllib.classification.{SVMModel, SVMWithSGD}
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import java.util.Properties
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row
```

2. 读取训练数据
首先，读取训练文本文件；然后，通过map将每行的数据用“,”隔开，在数据集中，每行被分成了5部分，前4部分是用户交易的3个特征(age_range,gender,merchant_id)，最后一部分是用户交易的分类(label)。把这里我们用LabeledPoint来存储标签列和特征列。LabeledPoint在监督学习中常用来存储标签和特征，其中要求标签的类型是double，特征的类型是Vector。
```
val train_data = sc.textFile("/dbtaobao/dataset/train_after.csv")
val test_data = sc.textFile("/dbtaobao/dataset/test_after.csv")
```

3. 构建模型
```
val train= train_data.map{line =>
  val parts = line.split(',')
  LabeledPoint(parts(4).toDouble,Vectors.dense(parts(1).toDouble,parts
(2).toDouble,parts(3).toDouble))
}
val test = test_data.map{line =>
  val parts = line.split(',')
  LabeledPoint(parts(4).toDouble,Vectors.dense(parts(1).toDouble,parts(2).toDouble,parts(3).toDouble))
}
```
接下来，通过训练集构建模型SVMWithSGD。这里的SGD即著名的随机梯度下降算法（Stochastic Gradient Descent）。设置迭代次数为1000，除此之外还有stepSize（迭代步伐大小），regParam（regularization正则化控制参数），miniBatchFraction（每次迭代参与计算的样本比例），initialWeights（weight向量初始值）等参数可以进行设置。
```
val numIterations = 1000
val model = SVMWithSGD.train(train, numIterations)
```

4. 评估模型
接下来，我们清除默认阈值，这样会输出原始的预测评分，即带有确信度的结果。
```
model.clearThreshold()
val scoreAndLabels = test.map{point =>
  val score = model.predict(point.features)
  score+" "+point.label
}
scoreAndLabels.foreach(println)
```
如果我们设定了阀值，则会把大于阈值的结果当成正预测，小于阈值的结果当成负预测。
```
model.setThreshold(0.0)
scoreAndLabels.foreach(println)
```
> 输出结果最后10条数据如下:

-36145.25547982101 1.0

-41436.42877491983 1.0

-93968.91068332111 1.0

-27617.286596836053 1.0

-91757.8479348277 1.0

-34526.44168181691 1.0

-25860.69738111214 1.0

-29591.03376107271 1.0

-40863.08944799144 1.0

-14765.484371218148 1.0

5. 把结果添加到mysql数据库中
现在我们上面没有设定阀值的测试集结果存入到MySQL数据中。
```
model.clearThreshold()
val scoreAndLabels = test.map{point =>
  val score = model.predict(point.features)
  score+" "+point.label
}
//设置回头客数据
val rebuyRDD = scoreAndLabels.map(_.split(" "))
/下面要设置模式信息
val schema = StructType(List(StructField("score", StringType, true),StructField("label", StringType, true)))
//下面创建Row对象，每个Row对象都是rowRDD中的一行
val rowRDD = rebuyRDD.map(p => Row(p(0).trim, p(1).trim))
//建立起Row对象和模式之间的对应关系，也就是把数据和模式对应起来
val rebuyDF = spark.createDataFrame(rowRDD, schema)
//下面创建一个prop变量用来保存JDBC连接参数
val prop = new Properties()
prop.put("user", "root") //表示用户名是root
prop.put("password", "123456") //表示密码是123456
prop.put("driver","com.mysql.jdbc.Driver") //表示驱动程序是com.mysql.jdbc.Driver
//下面就可以连接数据库，采用append模式，表示追加记录到数据库dbtaobao的rebuy表中
rebuyDF.write.mode("append").jdbc("jdbc:mysql://localhost:3306/dbtaobao", "dbtaobao.rebuy", prop)
```
退出scala shell
```
scala> :quit
```

6. 在mysql中查看导出的结果
```
# mysql -u root -p
mysql> use dbtaobao;
mysql> select * from rebuy limit 10;
```

到这里，第四个步骤的实验内容顺利结束。