---
layout: global
title: Spark编程指南
description: Spark SPARK_VERSION_SHORT programming guide in Java, Scala and Python
---

* This will become a table of contents (this text will be scraped).
{:toc}


# 概览

从高层面来讲，每一个Spark应用都由一个用于运行用户的`main`函数并在一个集群上执行*并行操作*的*驱动程序*构成。
Spark所提供的最主要的抽象是一个*弹性分布式数据集*(RDD)，它是一组分布在集群中多个节点上可以被并行操作的元素集。
RDD可以来自于一个Hadoop文件系统中的文件(或任何Hadoop支持的文件系统)，或是一个已存在于驱动程序中的Scala集合转换而来。
用户可以通过Spark将RDD*持久化*于内从中，从而可以使其在并行操作中高效的重用。而且RDD在节点失效后自动回复。

Spark所提供的第二个抽象是一组可被用于并行操作的*共享的变量*。默认情况下，当Spark在不同的节点上并行的将一个函数作为一组任务运行时，
它会将该函数中使用的每一个变量复制并携带进入各个任务中。有时一个变量需要被在多个任务中共享，或在任务与驱动程序之间共享。
Spark支持两类共享变量：用于在所有节点内存中缓存值的*广播变量*，以及*累加变量*--仅被用于进行"累加"操作，例如计数器和求和。

这篇指南将在Spark所支持的语言中展示以上特性。跟进的最简单的方式就是启动并使用Spark的交互式shell--无论是scala形式的`bin/spark-shell`还是Python
形式的`bin/pyspark`。

# 连接Spark

<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark {{site.SPARK_VERSION}}使用了Scala {{site.SCALA_BINARY_VERSION}}。如果打算用Scala来写应用，
你需要使用一个相兼容的Scala版本(比如{{site.SCALA_BINARY_VERSION}}.X)。

如果打算编写Spark应用，你还需要将Spark添加进Maven依赖中。Spark依赖可以在Maven中央库中找到：

    groupId = org.apache.spark
    artifactId = spark-core_{{site.SCALA_BINARY_VERSION}}
    version = {{site.SPARK_VERSION}}

另外，如果你打算访问HDFS集群，你还需要为你的HDFS版本添加`hadoop-client`依赖。
[third party distributions](hadoop-third-party-distributions.html)罗列了一些通用的HDFS版本标签。

    groupId = org.apache.hadoop
    artifactId = hadoop-client
    version = <your-hdfs-version>

最后，你需要将一些Spark类导入你的程序中。加入以下代码：

{% highlight scala %}
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
{% endhighlight %}

(在Spark 1.3.0之前，你需要显示的导入`import org.apache.spark.SparkContext._`来启用一些基本的隐式类型转换)

</div>

<div data-lang="java"  markdown="1">

Spark {{site.SPARK_VERSION}}需要Java 7或更高版本。如果你使用Java 8，Spark还支持用
[lambda表达式](http://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
来更简洁的书写函数，否则的话你需要使用在
[org.apache.spark.api.java.function](api/java/index.html?org/apache/spark/api/java/function/package-summary.html)
包下面的类。

如果打算用Java编写Spark应用，你需要将Spark添加进Maven依赖中。Spark依赖可以在Maven中央库中找到：

    groupId = org.apache.spark
    artifactId = spark-core_{{site.SCALA_BINARY_VERSION}}
    version = {{site.SPARK_VERSION}}

另外，如果你打算访问HDFS集群，你还需要为你的HDFS版本添加`hadoop-client`依赖。
[third party distributions](hadoop-third-party-distributions.html)罗列了一些通用的HDFS版本标签。

    groupId = org.apache.hadoop
    artifactId = hadoop-client
    version = <your-hdfs-version>

最后，你需要将一些Spark类导入你的程序中。加入以下代码：

{% highlight scala %}
import org.apache.spark.api.java.JavaSparkContext
import org.apache.spark.api.java.JavaRDD
import org.apache.spark.SparkConf
{% endhighlight %}

</div>

<div data-lang="python"  markdown="1">

Spark {{site.SPARK_VERSION}}需要Python 2.6+或Python 3.4+。Spark可以使用标准的CPython解释器，
因而像NumPy一类的C库可以被使用。同样的，Spark也可以使用PyPy 2.3+。

如果想使用Python启动Spark应用，请使用`bin/spark-submit`脚本。该脚本会将Spark Java/Scala库导入，并允许你
将应用提交到一个集群中。你可以使用`bin/pyspark`来启动交互式Python shell。

如果你打算访问HDFS数据，你需要使用一个与你的HDFS版本相关的PySpark版本库。
[third party distributions](hadoop-third-party-distributions.html)罗列了一些通用的HDFS版本标签。
Spark首页也罗列了通用HDFS版本可以使用的[预构建包](http://spark.apache.org/downloads.html)

最后，你需要将一些Spark类导入你的程序中。加入以下代码：

{% highlight python %}
from pyspark import SparkContext, SparkConf
{% endhighlight %}

PySpark需要在driver和workers中使用相同小版本的Python。它使用了PATH变量中声明的默认的python，你也可以通过声明`PYSPARK_PYTHON`
来指明你所想使用的Python的版本，例如：

{% highlight bash %}
$ PYSPARK_PYTHON=python3.4 bin/pyspark
$ PYSPARK_PYTHON=/opt/pypy-2.5/bin/pypy bin/spark-submit examples/src/main/python/pi.py
{% endhighlight %}

</div>

</div>


# 初始化Spark

<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark程序首先要做的事情就是创建[SparkContext](api/scala/index.html#org.apache.spark.SparkContext)对象, 它可以通知
Spark如何访问一个集群。为了创建`SparkContext`对象，你首先要建立一个包含你的应用程序信息的
[SparkConf](api/scala/index.html#org.apache.spark.SparkConf)对象。

每个JVM中应当只有一个活跃的SparkContext。在创建一个新的之前，你需要`stop()`当前活跃的SparkContext。

{% highlight scala %}
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

Spark程序首先要做的事情就是创建[JavaSparkContext](api/java/index.html?org/apache/spark/api/java/JavaSparkContext.html)
对象，它可以通知Spark如何访问一个集群。为了创建`SparkContext`对象，你首先要建立一个包含你的应用程序信息的
[SparkConf](api/java/index.html?org/apache/spark/SparkConf.html)对象。

{% highlight java %}
SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
JavaSparkContext sc = new JavaSparkContext(conf);
{% endhighlight %}

</div>

<div data-lang="python"  markdown="1">

Spark程序首先要做的事情就是创建[SparkContext](api/python/pyspark.html#pyspark.SparkContext)对象，
它可以通知Spark如何访问一个集群。为了创建`SparkContext`对象，你首先要建立一个包含你的应用程序信息的
[SparkConf](api/python/pyspark.html#pyspark.SparkConf)对象。

{% highlight python %}
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
{% endhighlight %}

</div>

</div>

参数`appName`表示的是将来在集群UI中显示的你的程序的名字。参数`master`则是一个
[Spark，Mesos或者YARN集群的URL](submitting-applications.html#master-urls)，
或是一个特殊的"local"字符串用于运行本地模式。在实际应用中，当运行于一个集群中时，
你不会想去硬编码一个`master`到你的程序中的，取而代之的则是
[使用`spark-submit`启动程序](submitting-applications.html)接收该参数。
但是在本地测试或单元测试时，你可以传递"local"来在本地进程中运行Spark。


## 使用Spark shell

<div class="codetabs">

<div data-lang="scala"  markdown="1">

在Spark shell启动时，一个可以被解释器所感知的SparkContext会被同时创建，并存放于变量`sc`中。这时你自行创建的SparkContext将无法使用。
你可以在启动时通过`--master`参数来指定context所要连接的master实例，并且还可以通过`--jars`参数来指定你所打算加载入类路径的jar包(多个
的话使用逗号来分割)。你还可以通过`--packages`参数后加maven坐标(多个的话使用逗号分割)的方式来往shell会话中添加对应的依赖项(例如对应的
Spark包)。如果需要添加依赖项目所在的maven仓库(比如SonaType)，可以通过`--repositories`参数来添加。例如，如果打算在本地的四个CPU核
上运行`bin/spark-shell`，可以用如下方式：

{% highlight bash %}
$ ./bin/spark-shell --master local[4]
{% endhighlight %}

如果打算在类路径里添加`code.jar`，可以用如下方式：

{% highlight bash %}
$ ./bin/spark-shell --master local[4] --jars code.jar
{% endhighlight %}

如果打算通过maven坐标的方式添加依赖：

{% highlight bash %}
$ ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"
{% endhighlight %}

请使用`spark-shell --help`来查看完整的命令选项。事实上在幕后，`spark-shell`调用了更多通用的
[`spark-submit`脚本](submitting-applications.html).

</div>

<div data-lang="python"  markdown="1">

在PySpark shell启动时，一个可以被解释器所感知的SparkContext会被同时创建，并存放于变量`sc`中。这时你自行创建的SparkContext将无法使用。
你可以在启动时通过`--master`参数来指定context所要连接的master实例，并且还可以通过`--py-files`参数来指定你所打算加载入运行时的python包
(.zip，.egg或者.py文件，多个的话使用逗号来分割)。你还可以通过`--packages`参数后加maven坐标(多个的话使用逗号分割)的方式来往shell会话中
添加对应的依赖项(例如对应的Spark包)。如果需要添加依赖项目所在的maven仓库(比如SonaType)，可以通过`--repositories`参数来添加。
当需要时，任何Spark包所要依赖的python依赖(在spark包的requirements.txt文件中有列出)需要手动的通过pip来进行安装。例如，如果打算在本地的
四个CPU核上运行`bin/spark-shell`，可以用如下方式：

{% highlight bash %}
$ ./bin/pyspark --master local[4]
{% endhighlight %}

如果打算在搜索路径里添加`code.py`(用以在后边`import code`)，可以用如下方式：

{% highlight bash %}
$ ./bin/pyspark --master local[4] --py-files code.py
{% endhighlight %}

请使用`pyspark --help`来查看完整的命令选项。事实上在幕后，`pyspark`调用了更多通用的
[`spark-submit`脚本](submitting-applications.html).

你也可以使用一个叫做[IPython](http://ipython.org)的增强版Python解释器来启动PySpark Shell。PySpark可以
在IPython1.0.0及以上版本下工作。可以在运行`bin/pyspark`时通过设置`PYSPARK_DRIVER_PYTHON`环境变量来使用IPython。

{% highlight bash %}
$ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark
{% endhighlight %}

你可以通过设置`PYSPARK_DRIVER_PYTHON_OPTS`来定制化`ipython`命令。例如，启动一个带有PyLab plot支持的
[IPython Notebook](http://ipython.org/notebook.html)：

{% highlight bash %}
$ PYSPARK_DRIVER_PYTHON=ipython PYSPARK_DRIVER_PYTHON_OPTS="notebook" ./bin/pyspark
{% endhighlight %}

在IPython Notebook服务器启动后，你可以在"Files"标签下创建一个新的的"Python 2" notebook。在notebook中，
你可以在试用Spark之前通过输入`%pylab inline`来使用其作为notebook的一部分。

</div>

</div>

# 弹性分布式数据集(RDDs)

Spark整体上是围绕着一个叫做 _弹性分布式数据集_ (RDD)的概念来展开的。它是一个可以并行操作并可以容错的元素集合。
我们可以使用两种方式来创建RDD：*并行化(parallelizing)* 一个驱动程序中的已存集合，或者引用一个外部存储系统中的数据集合，
比如一个共享的文件系统，HDFS，HBase，或者任何可以提供Hadoop InputFormat的数据源。

## 并行化的集合

<div class="codetabs">

<div data-lang="scala"  markdown="1">

并行化集合可以通过在你的驱动程序内的一个已存集合上(例如一个Scala的`Seq`)调用`SparkContext`的`parallelize`方法来获得。
集合内的元素会被拷贝并构成一个能够并行化操作的分布式数据集。例如以下代码展示了如何创建一个包含1到5的并行化集合：

{% highlight scala %}
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
{% endhighlight %}

一旦被创建，这个分布式数据集(`distData`)就可以并行化操作了。例如，我们可以调用`distData.reduce((a,b) => a + b)`来将数组元素
进行累加。我们将在后面继续介绍分布式数据集的操作。

</div>

<div data-lang="java"  markdown="1">

并行化集合可以通过在你的驱动程序内的一个已存集合上调用`JavaSparkContext`的`parallelize`方法来获得。
集合内的元素会被拷贝并构成一个能够并行化操作的分布式数据集。例如以下代码展示了如何创建一个包含1到5的并行化集合：

{% highlight java %}
List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
JavaRDD<Integer> distData = sc.parallelize(data);
{% endhighlight %}

一旦被创建，这个分布式数据集(`distData`)就可以并行化操作了。例如，我们可以调用`distData.reduce((a, b) -> a + b)`来将列表元素
进行累加。我们将在后面继续介绍分布式数据集的操作。

**注意：** *在本指南中，我们会经常使用Java 8 lambda语法来表示Java函数，如果你使用旧版本的Java，你可以通过实现
[org.apache.spark.api.java.function](api/java/index.html?org/apache/spark/api/java/function/package-summary.html)
包内的接口来达到同样的目的。我们将在后面更详细的介绍[如何给Spark传递函数](#passing-functions-to-spark)

</div>

<div data-lang="python"  markdown="1">

并行化集合可以通过在你的驱动程序内的一个已存集合或迭代器上调用`SparkContext`的`parallelize`方法来获得。
集合内的元素会被拷贝并构成一个能够并行化操作的分布式数据集。例如以下代码展示了如何创建一个包含1到5的并行化集合：

{% highlight python %}
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
{% endhighlight %}

一旦被创建，这个分布式数据集(`distData`)就可以并行化操作了。例如，我们可以调用`distData.reduce(lambda a, b: a + b)`来将列表元素
进行累加。我们将在后面继续介绍分布式数据集的操作。

</div>

</div>

并行化集合的一个重要参数就是 *partitions* --用以分割数据集。Spark将在集群中的每一个分区上运行一个任务。
一般情况下你可以在你的及群众为每一个CPU分配2-4个分区。一般来说，Spark将会基于你的集群来自动的设置分区数量。
但你也可以通过`parallelize`方法的第二个参数来手动的设置(例如，`sc.parallelize(data,10)`)。
注意：代码中有些地方会使用slices(partitions的一个同义词)来保证向后兼容性。

## 外部数据集

<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark可以从任何支持Hadoop的外部存储源中创建分布式数据集，这些源包括你本地的文件系统，HDFS，Cassandra，HBase，
[Amazon S3](http://wiki.apache.org/hadoop/AmazonS3)等等。Spark同样也支持文本文件，
[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)
以及任何其他的Hadoop [InputFormat](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html)。

可以使用`SparkContext`的`textFile`方法来创建文本文件RDD。这个方法接受一个文件的URI(本地文件路径或`hdfs://`，`s3n://`，或其他合法URI)
并且将其构造成有文件文本行组成的集合。以下是一个样例调用：

{% highlight scala %}
scala> val distFile = sc.textFile("data.txt")
distFile: RDD[String] = MappedRDD@1d4cee08
{% endhighlight %}

一旦被创建，`distFile`就可以被用作数据集进行操作了。例如，我们可以通过`map`和`reduce`操作将所有行的长度进行累加：
`distFile.map(s => s.length).reduce((a, b) => a + b)`。

使用Spark读取文件需要注意：

* 如果使用一个本地文件系统的路径，文件需要在所有的worker结点上都能通过相同路径访问到。无论是通过拷贝文件到所有节点还是使用网络挂载的共享文件系统。

* 所有基于文件的Spark输入方法，包括`textFile`，都支持运行在目录，压缩文件以及通配符上。例如，你可以使用`textFile("/my/directory")`，`textFile("/my/directory/*.txt")`以及`textFile("/my/directory/*.gz")`来访问文件。

* `textFile`方法也可以接受一个可选的参数来控制文件的分区。默认情况下，Spark会为每个文件块分配一个分区(HDFS下文件块默认大小为64M)，但你可以通过传递一个比较大的值来请求分配更多的分区。需要注意的是你不能请求比文件块数目更少的分区。

除了文本文件外，Spark Scala API也支持其他几种数据格式：

* `SparkContext.wholeTextFiles`允许你读取一个包含很多小文本文件的目录，并且以(文件名，文件内容)元组的方式返回每个文件。与`textFile`相反，`textFile`会将每个文件中的每一行作为一个记录来返回。

* 对于[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)，可以使用SparkContext的`sequenceFile[K, V]`方法，其中`K`是以文件类型做为的键值，`V`是以文件本身作为的值。它们应当是Hadoop的[Writable](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Writable.html)接口的子类，就像[IntWritable](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/IntWritable.html)和[Text](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Text.html)。另外，Spark允许你指定一些通用Writable原生类型；例如，`sequenceFile[Int, String]`将会自动读取IntWritable和Text。

* 对于其他Hadoop InputFormat，你可以使用`SparkContext.hadoopRDD`方法，它必须接受一个`JobConf`类型的参数，以及input format类，key类和value类。使用与设置一个具有输入源的Hadoop作业相同的方式来设置这些值。你也可以对基于"新"的MapReduce API(`org.apache.hadoop.mapreduce`)的InputFormat来使用`SparkContext.newAPIHadoopRDD`方法。

* `RDD.saveAsObjectFile`以及`SparkContext.objectFile`支持将由一个序列化的Java对象构成的简单格式存储为一个RDD。虽然这种方式不会像专属的格式(例如Avro)那样高效，但它提供了一种简单的方式来保存任意的RDD。

</div>

<div data-lang="java"  markdown="1">

Spark可以从任何支持Hadoop的外部存储源中创建分布式数据集，这些源包括你本地的文件系统，HDFS，Cassandra，HBase，
[Amazon S3](http://wiki.apache.org/hadoop/AmazonS3)等等。Spark同样也支持文本文件，
[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)
以及任何其他的Hadoop [InputFormat](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html)。

可以使用`SparkContext`的`textFile`方法来创建文本文件RDD。这个方法接受一个文件的URI(本地文件路径或`hdfs://`，`s3n://`，或其他合法URI)
并且将其构造成有文件文本行组成的集合。以下是一个样例调用：

{% highlight java %}
JavaRDD<String> distFile = sc.textFile("data.txt");
{% endhighlight %}

一旦被创建，`distFile`就可以被用作数据集进行操作了。例如，我们可以通过`map`和`reduce`操作将所有行的长度进行累加：
`distFile.map(s -> s.length()).reduce((a, b) -> a + b)`。

使用Spark读取文件需要注意：

* 如果使用一个本地文件系统的路径，文件需要在所有的worker结点上都能通过相同路径访问到。无论是通过拷贝文件到所有节点还是使用网络挂载的共享文件系统。

* 所有基于文件的Spark输入方法，包括`textFile`，都支持运行在目录，压缩文件以及通配符上。例如，你可以使用`textFile("/my/directory")`，`textFile("/my/directory/*.txt")`以及`textFile("/my/directory/*.gz")`来访问文件。

* `textFile`方法也可以接受一个可选的参数来控制文件的分区。默认情况下，Spark会为每个文件块分配一个分区(HDFS下文件块默认大小为64M)，但你可以通过传递一个比较大的值来请求分配更多的分区。需要注意的是你不能请求比文件块数目更少的分区。

除了文本文件外，Spark Scala API也支持其他几种数据格式：

* `JavaSparkContext.wholeTextFiles`允许你读取一个包含很多小文本文件的目录，并且以(文件名，文件内容)元组的方式返回每个文件。与`textFile`相反，`textFile`会将每个文件中的每一行作为一个记录来返回。

* 对于[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)，可以使用SparkContext的`sequenceFile[K, V]`方法，其中`K`是以文件类型做为的键值，`V`是以文件本身作为的值。它们应当是Hadoop的[Writable](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Writable.html)接口的子类，就像[IntWritable](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/IntWritable.html)和[Text](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Text.html)。另外，Spark允许你指定一些通用Writable原生类型；例如，`sequenceFile[Int, String]`将会自动读取IntWritable和Text。

* 对于其他Hadoop InputFormat，你可以使用`JavaSparkContext.hadoopRDD`方法，它必须接受一个`JobConf`类型的参数，以及input format类，key类和value类。使用与设置一个具有输入源的Hadoop作业相同的方式来设置这些值。你也可以对基于"新"的MapReduce API(`org.apache.hadoop.mapreduce`)的InputFormat来使用`JavaSparkContext.newAPIHadoopRDD`方法。

* `JavaRDD.saveAsObjectFile`以及`JavaSparkContext.objectFile`支持将由一个序列化的Java对象构成的简单格式存储为一个RDD。虽然这种方式不会像专属的格式(例如Avro)那样高效，但它提供了一种简单的方式来保存任意的RDD。

</div>

<div data-lang="python"  markdown="1">

PySpark可以从任何支持Hadoop的外部存储源中创建分布式数据集，这些源包括你本地的文件系统，HDFS，Cassandra，HBase，
[Amazon S3](http://wiki.apache.org/hadoop/AmazonS3)等等。Spark同样也支持文本文件，
[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)
以及任何其他的Hadoop [InputFormat](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html)。

可以使用`SparkContext`的`textFile`方法来创建文本文件RDD。这个方法接受一个文件的URI(本地文件路径或`hdfs://`，`s3n://`，或其他合法URI)
并且将其构造成有文件文本行组成的集合。以下是一个样例调用：

{% highlight python %}
>>> distFile = sc.textFile("data.txt")
{% endhighlight %}

一旦被创建，`distFile`就可以被用作数据集进行操作了。例如，我们可以通过`map`和`reduce`操作将所有行的长度进行累加：
`distFile.map(lambda s: len(s)).reduce(lambda a, b: a + b)`。

使用Spark读取文件需要注意：

* 如果使用一个本地文件系统的路径，文件需要在所有的worker结点上都能通过相同路径访问到。无论是通过拷贝文件到所有节点还是使用网络挂载的共享文件系统。

* 所有基于文件的Spark输入方法，包括`textFile`，都支持运行在目录，压缩文件以及通配符上。例如，你可以使用`textFile("/my/directory")`，`textFile("/my/directory/*.txt")`以及`textFile("/my/directory/*.gz")`来访问文件。

* `textFile`方法也可以接受一个可选的参数来控制文件的分区。默认情况下，Spark会为每个文件块分配一个分区(HDFS下文件块默认大小为64M)，但你可以通过传递一个比较大的值来请求分配更多的分区。需要注意的是你不能请求比文件块数目更少的分区。

除了文本文件外，Spark Scala API也支持其他几种数据格式：

* `SparkContext.wholeTextFiles`允许你读取一个包含很多小文本文件的目录，并且以(文件名，文件内容)元组的方式返回每个文件。与`textFile`相反，`textFile`会将每个文件中的每一行作为一个记录来返回。

* `RDD.saveAsObjectFile`以及`SparkContext.objectFile`支持将由pickled化的python对象构成的简单格式存储为一个RDD。批处理用于pickle序列化，默认的批处理数量为10。

* SequenceFile 以及 Hadoop Input/Output Format

**注意** 该功能目前被标记为```实验性```，并且仅仅面向高级用户。它很有可能会在将来被基于Spark SQL的读/写支持所替换，并且在这种情况下Spark SQL会是优先推荐使用的方式。

**Writable支持**

PySpark SequenceFile支持从一个Java的键值对中装载一个RDD，支持将Writable转换为基本的Java类型，同样也支持使用
[Pyrolite](https://github.com/irmen/Pyrolite/)来将生成的Java对象进行pickle化。当将一个键值对构成的RDD保存为一个SequenceFile时，PySpark
可以做相反的操作。它可以将Python对象反pickle化为Java对象，然后将之转化为Writable。以下的Writable可以被自动转化：

<table class="table">
<tr><th>Writable Type</th><th>Python Type</th></tr>
<tr><td>Text</td><td>unicode str</td></tr>
<tr><td>IntWritable</td><td>int</td></tr>
<tr><td>FloatWritable</td><td>float</td></tr>
<tr><td>DoubleWritable</td><td>float</td></tr>
<tr><td>BooleanWritable</td><td>bool</td></tr>
<tr><td>BytesWritable</td><td>bytearray</td></tr>
<tr><td>NullWritable</td><td>None</td></tr>
<tr><td>MapWritable</td><td>dict</td></tr>
</table>

数组无法开箱即用。当进行读写时用户需要指定自定义的`ArrayWritable`子类型。当写入时，用户还需要指定自定义的转换器来将数组转换为自定义的`ArrayWritable`字类型。
当读取时，默认的转换器会将自定义的`ArrayWritable`字类型转换为Java的`Object[]`，然后被pickle化为Python的tuple。
为了得到Python的原生数据类型的数组的`array.array`，用户需要自定义转换器。

**保存和读取SequenceFiles**

与文本文件相类似，SequenceFiles可以通过指定路径来被保存和读取。key和value类可以被指定，但对标准的Writable来说这些并不需要。

{% highlight python %}
>>> rdd = sc.parallelize(range(1, 4)).map(lambda x: (x, "a" * x ))
>>> rdd.saveAsSequenceFile("path/to/file")
>>> sorted(sc.sequenceFile("path/to/file").collect())
[(1, u'a'), (2, u'aa'), (3, u'aaa')]
{% endhighlight %}

**保存和读取其他Hadoop Input/Output Format**

PySpark同样可以读取任意的Hadoop InputFormat以及写入任意的Hadoop OutputFormat，不管你使用的是`新`的还是`旧`的Hadoop MapReduce API。
如果需要的话，一个Hadoop配置可以以一个Python dict的方式被传入。这里就是一个使用Elasticsearch ESInputFormat的例子：

{% highlight python %}
$ SPARK_CLASSPATH=/path/to/elasticsearch-hadoop.jar ./bin/pyspark
>>> conf = {"es.resource" : "index/type"}   # assume Elasticsearch is running on localhost defaults
>>> rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",\
    "org.apache.hadoop.io.NullWritable", "org.elasticsearch.hadoop.mr.LinkedMapWritable", conf=conf)
>>> rdd.first()         # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
{% endhighlight %}

需要注意的是，如果InputFormat仅仅是简单的依赖与一个Hadoop配置和/或一个输入路径，并且key和value类可以很容易的按照上面的表哥进行转化的话，
那么这个方法对这类情形来说可以很好的工作。

如果你有自定义序列化的二进制数据(比如从Cassandra/HBase中加载的数据)，那么你需要首先将其转换为可以被Pyrolite pickler处理的Scala/Java类型。
[Converter](api/scala/index.html#org.apache.spark.api.python.Converter)可以用来做这类事情。只需要扩展这个trait然后在```convert```
方法里实现你的转换代码即可。但要注意保证用以访问你的```InputFormat```的该类以及其需要的依赖要打入你的Spark job jar中，并被引入PySpark的类路径里。

如果想查看使用Cassandra / HBase ```InputFormat``` 和 ```OutputFormat```自定义转换器的例子，请看
[Python examples]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python)以及
[Converter examples]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples/pythonconverters)。

</div>
</div>

## RDD操作

RDD支持两类操作：*transformations*，用以从一个已存的数据集中创建一个新的数据集，以及 *actions*，用以在一个数据集上执行了一系列计算后来往驱动程序中返回一个值。
例如，`map`操作就是一个transformation操作，该操作通过将每个数据集元素传递给一个函数然后返回一个新的RDD来容纳结果。另一方面，`reduce`则是一个`action`操作，该
操作将会通过一个函数来对RDD元素进行聚集操作，并将最终结果返回给驱动程序(尽管同样也存在一个并行的`reduceByKey`操作来返回一个分布式数据集)。

Spark中所有transformation都是<i>懒惰的</i>，意味着它们不会马上来计算结果。相反，它们仅仅是记录下来对基础数据集(例如一个文件)的transformation操作。
transformation操作仅当驱动程序的action操作需要返回一个结果的时候才会去进行计算。这种设计可以让Spark更高效的运行 -- 例如，我们应该更倾向于
通过`map`操作创建的一个数据集被用于一个`reduce`操作，然后仅仅将`reduce`操作的结果返回给驱动程序，而不是去获得一个非常大的map数据集。

默认情况下，每个转换后(transformed)的RDD可以通过重复的在其上运行action来进行重复计算。但是你也可以通过`persist`(或`cache`)方法将RDD`持久化`到内存中，
在这种情况下，Spark将会在集群中保留这些元素以便允许你下次查询时非常快的获得到它们。同样的，将RDD持久化到磁盘上或在多个集群节点中进行复制也是支持的。

### 基础

<div class="codetabs">

<div data-lang="scala" markdown="1">

为了展示RDD的基础概念，先让我们看一个简单的程序：

{% highlight scala %}
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
{% endhighlight %}

第一行使用外部文件定义了一个基本的RDD。这个数据集并没有真正的被载入内存或其他可用设备中：`lines`仅仅是一个该文件的指针而已。
第二行将一个`map`转换的结果定义为一个`lineLengths`的变量。同样的，`lineLengths`并*不是*被立即被计算的，而是要作为延迟计算的。
最后，我们运行一个`reduce`操作。在这时Spark将被实际计算工作分拆为运行在多个机器上的子任务，然后每台机器自行运算其分配的一部分map
计算和其本地的规约(reduction)计算，最后仅将其自己的计算结果返回给驱动程序。

如果我们同样想在后续计算中使用`lineLengths`，我们可以在`reduce`前添加以下代码：

{% highlight scala %}
lineLengths.persist()
{% endhighlight %}

这将导致`lineLengths`在其第一次被计算后被保存到内存中。

</div>

<div data-lang="java" markdown="1">

为了展示RDD的基础概念，先让我们看一个简单的程序：

{% highlight java %}
JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(s -> s.length());
int totalLength = lineLengths.reduce((a, b) -> a + b);
{% endhighlight %}

第一行使用外部文件定义了一个基本的RDD。这个数据集并没有真正的被载入内存或其他可用设备中：`lines`仅仅是一个该文件的指针而已。
第二行将一个`map`转换的结果定义为一个`lineLengths`的变量。同样的，`lineLengths`并*不是*被立即被计算的，而是要作为延迟计算的。
最后，我们运行一个`reduce`操作。在这时Spark将被实际计算工作分拆为运行在多个机器上的子任务，然后每台机器自行运算其分配的一部分map
计算和其本地的规约(reduction)计算，最后仅将其自己的计算结果返回给驱动程序。

如果我们同样想在后续计算中使用`lineLengths`，我们可以在`reduce`前添加以下代码：

{% highlight java %}
lineLengths.persist(StorageLevel.MEMORY_ONLY());
{% endhighlight %}

这将导致`lineLengths`在其第一次被计算后被保存到内存中。

</div>

<div data-lang="python" markdown="1">

为了展示RDD的基础概念，先让我们看一个简单的程序：

{% highlight python %}
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
{% endhighlight %}

第一行使用外部文件定义了一个基本的RDD。这个数据集并没有真正的被载入内存或其他可用设备中：`lines`仅仅是一个该文件的指针而已。
第二行将一个`map`转换的结果定义为一个`lineLengths`的变量。同样的，`lineLengths`并*不是*被立即被计算的，而是要作为延迟计算的。
最后，我们运行一个`reduce`操作。在这时Spark将被实际计算工作分拆为运行在多个机器上的子任务，然后每台机器自行运算其分配的一部分map
计算和其本地的归约(reduction)计算，最后仅将其自己的计算结果返回给驱动程序。

如果我们同样想在后续计算中使用`lineLengths`，我们可以在`reduce`前添加以下代码：

{% highlight python %}
lineLengths.persist()
{% endhighlight %}

这将导致`lineLengths`在其第一次被计算后被保存到内存中。

</div>

</div>

### 向Spark传递函数

<div class="codetabs">

<div data-lang="scala" markdown="1">

Spark API重度依赖于向驱动程序中传递函数用以在集群中来运行。建议使用以下两种方式来这样做：

* [匿名函数(Anonymous function syntax](http://docs.scala-lang.org/tutorials/tour/anonymous-function-syntax.html)，可以用在小段代码中。
* 在全局单例对象(global singleton object)中使用静态方法。比如，你可以定义`ojbect MyFunctions`以及对应函数，然后将函数传入，就像下面这样：

{% highlight scala %}
object MyFunctions {
  def func1(s: String): String = { ... }
}

myRdd.map(MyFunctions.func1)
{% endhighlight %}

注意同样有可能将一个类(区别于一个单例对象)中的函数引用传递给Spark，但这需要将该类的实际对象及其函数引用传入。例如以下这样：

{% highlight scala %}
class MyClass {
  def func1(s: String): String = { ... }
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
}
{% endhighlight %}

这里，如果我们使用`new MyClass`创建一个对象然后调用它的`doStuff`函数，那么里面的`map`将会引用到*这个`MyClass`对象实例*的`func1`方法，这样
的话整个对象需要被发送到集群中。这跟写`rdd.map(x => this.func1(x))`是类似的。

类似的，访问对象的成员变量也将会引用整个对象：

{% highlight scala %}
class MyClass {
  val field = "Hello"
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
}
{% endhighlight %}

这段代码与`rdd.map(x => this.field + x)`相同，它们都会引用整个`this`对象。如果想避免这种情况，最简单的方法就是拷贝`field`到一个本地变量来使用，
而不是直接使用它：

{% highlight scala %}
def doStuff(rdd: RDD[String]): RDD[String] = {
  val field_ = this.field
  rdd.map(x => field_ + x)
}
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

Spark API重度依赖于向驱动程序中传递函数用以在集群中来运行。在Java中，函数式接口需要通过实现
[org.apache.spark.api.java.function](api/java/index.html?org/apache/spark/api/java/function/package-summary.html)包下的接口来实现。
有下两种方式来创建函数

* 让你的类实现Function接口，无论是匿名内部类还是命名的类，让后将其实例传递Spark。
* 在Java 8下，使用[lambda表达式](http://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)来简洁的定义一个函数实现。

虽然这篇指南中使用了lambda语法来更简洁的表示，但使用相同的长格式(long-form)API方式来实现也很容易。比如，我们可以像以下这样的方式来书写代码：

{% highlight java %}
JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(new Function<String, Integer>() {
  public Integer call(String s) { return s.length(); }
});
int totalLength = lineLengths.reduce(new Function2<Integer, Integer, Integer>() {
  public Integer call(Integer a, Integer b) { return a + b; }
});
{% endhighlight %}

或者，如果使用内联函数比较笨拙的话，可以这样：

{% highlight java %}
class GetLength implements Function<String, Integer> {
  public Integer call(String s) { return s.length(); }
}
class Sum implements Function2<Integer, Integer, Integer> {
  public Integer call(Integer a, Integer b) { return a + b; }
}

JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(new GetLength());
int totalLength = lineLengths.reduce(new Sum());
{% endhighlight %}

需要注意的是Java的匿名内部类可以访问被标注为`final`的在其外部的变量。Spark将会把这些变量的拷贝传递给每个工作节点，就像其他语言那样。

</div>

<div data-lang="python"  markdown="1">

Spark API重度依赖于向驱动程序中传递函数用以在集群中来运行。建议使用以下三种方式来这样做：

* [Lambda表达式](https://docs.python.org/2/tutorial/controlflow.html#lambda-expressions)，一种可以用来声明简单函数的表达式。
  (Lambda不支持多语句的函数或者不返回值的函数。)
* Local `def`s inside the function calling into Spark, for longer code.
* 模块内的高级函数。

例如，为了传递一个支持使用`lambda`的长函数，可以这样做：

{% highlight python %}
"""MyScript.py"""
if __name__ == "__main__":
    def myFunc(s):
        words = s.split(" ")
        return len(words)

    sc = SparkContext(...)
    sc.textFile("file.txt").map(myFunc)
{% endhighlight %}

需要注意的是有可能将一个类(区别于一个单例对象)中的函数引用传递给Spark，但这需要将该类的实际对象及其函数引用传入。例如以下这样：

{% highlight python %}
class MyClass(object):
    def func(self, s):
        return s
    def doStuff(self, rdd):
        return rdd.map(self.func)
{% endhighlight %}

这里，如果我们使用`new MyClass`创建一个对象然后调用它的`doStuff`函数，那么里面的`map`将会引用到*这个`MyClass`对象实例*的`func`方法，这样
的话整个对象需要被发送到集群中。

类似的，访问对象的成员变量也将会引用整个对象：

{% highlight python %}
class MyClass(object):
    def __init__(self):
        self.field = "Hello"
    def doStuff(self, rdd):
        return rdd.map(lambda s: self.field + s)
{% endhighlight %}

如果想避免这种情况，最简单的方法就是拷贝`field`到一个本地变量来使用，
而不是直接使用它：

{% highlight python %}
def doStuff(self, rdd):
    field = self.field
    return rdd.map(lambda s: field + s)
{% endhighlight %}

</div>

</div>

### 理解闭包 <a name="ClosuresLink"></a>
Spark的难点之一就是如何理解运行在集群中的代码中的变量和方法的作用域和生命周期。在变量作用域外改变RDD的操作在Spark中是一个经常导致理解困难的地方。
在下面的例子中，我们将会看到使用`foreach()`来增加计数器值的代码，但同样的问题也会发生在其他操作中。

#### 示例

考虑以下简单的RDD元素累加操作，无论是否其执行在相同的JVM中,它的行为都会有很大的不同。
一个通用的例子则是在`local`模式下运行Spark(`--master=local[n]`)，而不是将其部署在集群中(e.g. 通过spark-submit部署至YARN):

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% highlight scala %}
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
{% endhighlight %}
</div>

<div data-lang="java"  markdown="1">
{% highlight java %}
int counter = 0;
JavaRDD<Integer> rdd = sc.parallelize(data); 

// Wrong: Don't do this!!
rdd.foreach(x -> counter += x);

println("Counter value: " + counter);
{% endhighlight %}
</div>

<div data-lang="python"  markdown="1">
{% highlight python %}
counter = 0
rdd = sc.parallelize(data)

# Wrong: Don't do this!!
rdd.foreach(lambda x: counter += x)

print("Counter value: " + counter)

{% endhighlight %}
</div>

</div>

#### 本地模式 vs. 集群模式

最主要的挑战在于以上代码的行为是无法预料的。在一个单一JVM中的本的模式下，以上代码会在RDD中累加所有值并将其存入**counter**变量。这是因为RDD和**counter**变量都共存于驱动节点上的同一内存空间内。

但是，在`cluster`模式下，所发生的一切将更为复杂，并且以上代码可能无法按预期来工作。为了执行作业，Spark将RDD的操作流程拆分为一个个的任务(task) - 每个任务会被一个执行器(executor)来执行。
在执行之前，Spark将对**闭包**进行计算。闭包就是那些必须对executor可见的用来在RDD上进行其计算的(本例中为`foreach()`)变量和方法。这类闭包会被序列化然后发送到每个executor中。
在`本地(local)`模式中，因为只有一个executor，所以闭包会被共享至所有情况下。但在其他模式下，每个在分离的工作节点上的executor都会拥有一份该闭包的拷贝，而不是共享该闭包。

因而在这种情形下，闭包内的变量仅仅是拷贝，所以当**counter**在`foreach`函数中被引用时，它不再是那个在驱动节点上的**counter**变量了。当然驱动节点的内存中仍然还有一个**counter**变量，
但其已经对其他executor不再可见了！那些executor只能够看到通过序列化的闭包里传递过来的变量的拷贝。因此，**counter**变量最终的值将仍然是零，因为所有在**counter**上的操作都在针对那些
序列化后的闭包中的值上进行。

为了保证在这些情形下这类行为的正确性，我们应当使用[`累加器(Accumulator)`](#AccumLink)。Spark的累加器专门用来保证在任务执行被分配到一个集群中的多个节点时能够安全的更新其中的变量。
在本指南的累加器一节会专门探讨它。

总的来说，闭包 - 就像循环或本地化定义的方法那样，不应当被用于改变某些全局的状态。Spark不会对那些闭包之外的对象引用的改变行为作出任何定义或保证。
一些这样干的代码或许能在本地模式下运行，但这只是侥幸的并且在分布式模式下基本不会按照预期来运行。如果需要对全局状态进行聚合运算，请使用累加器。

#### 打印RDD中的元素
另一个比较通用的情形是想要使用`rdd.foreach(println)`或`rdd.map(println)`打印出RDD中的元素。在单机中，它可以生成所预期的输出并打印出所有RDD中的元素。
但是
Another common idiom is attempting to print out the elements of an RDD using `rdd.foreach(println)` or `rdd.map(println)`。但是在`集群`模式下，
executor对`stdout`所执行的输出已经被重定向到了executor自身的`stdout`上，而不是驱动程序上。因而驱动程序上的`stdout`将不会对RDD有任何显示！如果想打印在
驱动程序上的所有元素，你可以使用`collect()`方法首先将RDD放回驱动程序节点上：`rdd.collect().foreach(println)`。这很可能会导致驱动程序内存溢出，因为
`collect()`方法会把整个集群内的RDD抓取到一个节点上；如果你仅仅是想打印RDD中的一小部分元素，更安全的方法是用`take()`：`rdd.take(100).foreach(println)`。

### 使用key-value键值对

<div class="codetabs">

<div data-lang="scala" markdown="1">

大部分的Spark操作对包含各种类型对象的RDD均可以使用，但有一小部分特殊的操作仅能够在包含key-value健值对的RDD上使用。最常见的就是分布式"shuffle"操作，
例如通过键来进行分组(grouping)或聚合(aggregating)。

在Scala里，这些操作可以直接用在含有[Tuple2](http://www.scala-lang.org/api/{{site.SCALA_VERSION}}/index.html#scala.Tuple2)
(语言内建的元组对象, 通过`(a, b)`可以简单的构建)对象的RDD上。[PairRDDFunctions](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)
类里的key-value键值对操作也同样可以自动的包装tuples RDD。

比如，以下代码在一个key-value键值对上使用`reduceByKey`操作来计算在文件中每一行文字出现次数:

{% highlight scala %}
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
{% endhighlight %}

我们同样也可以使用`counts.sortByKey()`，比如，将键值对按照字母排序，并在最后通过`counts.collect()`来将它们作为一组对象返回给主驱动程序。

**注意：**当在键值对中使用自定义对象作为键时，必须保证同时拥有自定义的`equals()`方法和相匹配的`hashCode()`方法。详细情况请参看[Object.hashCode()
Documentation](http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#hashCode())。

</div>

<div data-lang="java" markdown="1">

大部分的Spark操作对包含各种类型对象的RDD均可以使用，但有一小部分特殊的操作仅能够在包含key-value健值对的RDD上使用。最常见的就是分布式"shuffle"操作，
例如通过键来进行分组(grouping)或聚合(aggregating)。

在Java里，key-value键值对一般使用Scala标准库中的[scala.Tuple2](http://www.scala-lang.org/api/{{site.SCALA_VERSION}}/index.html#scala.Tuple2)
你可以简单的通过调用`new Tuple2(a, b)`来创建一个tuple，并且使用`tuple._1()`和`tuple._2()`来访问其成员。

key-value键值对成员的RDD一般使用[JavaPairRDD](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)类来表示。你可以通过特殊版本
的`map`操作来从JavaRDD构建JavaPairRDD，比如`mapToPair`和`flatMapToPair`。JavaPairRDD既有普通标准RDD函数，也包含特殊的key-value类函数。

比如，以下代码在一个key-value键值对上使用`reduceByKey`操作来计算在文件中每一行文字出现次数:

{% highlight scala %}
JavaRDD<String> lines = sc.textFile("data.txt");
JavaPairRDD<String, Integer> pairs = lines.mapToPair(s -> new Tuple2(s, 1));
JavaPairRDD<String, Integer> counts = pairs.reduceByKey((a, b) -> a + b);
{% endhighlight %}

我们同样也可以使用`counts.sortByKey()`，比如，将键值对按照字母排序，并在最后通过`counts.collect()`来将它们作为一组对象返回给主驱动程序。

**注意：**当在键值对中使用自定义对象作为键时，必须保证同时拥有自定义的`equals()`方法和相匹配的`hashCode()`方法。详细情况请参看[Object.hashCode()
Documentation](http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#hashCode())。

</div>

<div data-lang="python" markdown="1">

大部分的Spark操作对包含各种类型对象的RDD均可以使用，但有一小部分特殊的操作仅能够在包含key-value健值对的RDD上使用。最常见的就是分布式"shuffle"操作，
例如通过键来进行分组(grouping)或聚合(aggregating)。

在Python里，这些操作可以直接用在Python内建的tuple(例如`(1, 2)`)RDD上。只需简单的创建tuple然后调用你想要的操作即可。

比如，以下代码在一个key-value键值对上使用`reduceByKey`操作来计算在文件中每一行文字出现次数:

{% highlight python %}
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)
{% endhighlight %}

我们同样也可以使用`counts.sortByKey()`，比如，将键值对按照字母排序，并在最后通过`counts.collect()`来将它们作为一组对象返回给主驱动程序。

</div>

</div>


### 转换(Transformation)

下表列出了一些Spark中常用的转换操作。详情请参看RDD API文档
([Scala](api/scala/index.html#org.apache.spark.rdd.RDD),
 [Java](api/java/index.html?org/apache/spark/api/java/JavaRDD.html),
 [Python](api/python/pyspark.html#pyspark.RDD),
 [R](api/R/index.html))
以及键值对RDD函数文档
([Scala](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions),
 [Java](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html))

<table class="table">
<tr><th style="width:25%">Transformation</th><th>Meaning</th></tr>
<tr>
  <td> <b>map</b>(<i>func</i>) </td>
  <td> 返回一个新建的分布式数据集，构建该集合需要通过一个函数<i>func</i>来将数据源的每一个元素导入。 </td>
</tr>
<tr>
  <td> <b>filter</b>(<i>func</i>) </td>
  <td> 返回一个新建的分布式数据集，构建该集合需要通过函数<i>func</i>来过滤数据源元素，并将符合返回值为true的元素进行导入。</td>
</tr>
<tr>
  <td> <b>flatMap</b>(<i>func</i>) </td>
  <td> 类似于map，但每一个输入项可以被map到0个或多个输出项上。(这样的话<i>func</i>可以返回一个Seq而不是一个单个的值)。 </td>
</tr>
<tr>
  <td> <b>mapPartitions</b>(<i>func</i>) <a name="MapPartLink"></a> </td>
  <td> 类似于map，但会在RDD的每个分区partition(或块block)上独立的运行，所以当在类型T的RDD上运行时，<i>func</i>必须是
    Iterator&lt;T&gt; => Iterator&lt;U&gt;类型   </td>
</tr>
<tr>
  <td> <b>mapPartitionsWithIndex</b>(<i>func</i>) </td>
  <td> 类似于mapPartitions，但还提供了一个包含用来表示分区(partition)索引的整型值的<i>func</i>函数，所以当在类型T的RDD上运行时，<i>func</i>必须是
  (Int, Iterator&lt;T&gt;) => Iterator&lt;U&gt;类型。
  </td>
</tr>
<tr>
  <td> <b>sample</b>(<i>withReplacement</i>, <i>fraction</i>, <i>seed</i>) </td>
  <td> 根据<i>fraction</i>指定的精度，对数据进行小规模取样，可选择是否用随机数替换，<i>seed</i>用于指定随机数生成器的种子。 </td>
</tr>
<tr>
  <td> <b>union</b>(<i>otherDataset</i>) </td>
  <td> 返回一个包含当前数据集和参数<i>otherDataset</i>中所有元素的并集的新数据集。 </td>
</tr>
<tr>
  <td> <b>intersection</b>(<i>otherDataset</i>) </td>
  <td> 返回一个包含当前数据集和参数<i>otherDataset</i>中所有元素的交集的新数据集。 </td>
</tr>
<tr>
  <td> <b>distinct</b>([<i>numTasks</i>])) </td>
  <td> 返回一个包含当前数据集中所有不重复元素的新数据集。</td>
</tr>
<tr>
  <td> <b>groupByKey</b>([<i>numTasks</i>]) <a name="GroupByLink"></a> </td>
  <td> 在一个（K, V）对的数据集上调用，返回一个(K，Iterable&lt;V&gt;)对的数据集。<br />
       <b>注意：</b>如果你使用grouping来对每个key执行聚集操作(比如sum或average)，使用<code>reduceByKey</code>或<code>aggregateByKey</code>
       会获得更好的性能。</br>
       <b>注意：</b>默认情况下，并行任务数要依赖于父RDD的分区数目。你可以传入一个可选的numTasks参数来改变任务数。
  </td>
</tr>
<tr>
  <td> <b>reduceByKey</b>(<i>func</i>, [<i>numTasks</i>]) <a name="ReduceByLink"></a> </td>
  <td> 在一个(K，V)对的数据集上调用时，返回一个(K，V)对的数据集，使用指定的reduce函数<i>func</i>，将相同key的值聚合到一起。<i>func</i>应当是(V,V) => V 类型的。
  类似<code>groupByKey</code>，reduce任务个数是可以通过第二个可选参数来配置的。</td>
</tr>
<tr>
  <td> <b>aggregateByKey</b>(<i>zeroValue</i>)(<i>seqOp</i>, <i>combOp</i>, [<i>numTasks</i>]) <a name="AggregateByLink"></a> </td>
  <td> 在一个(K，V)对的数据集上调用时，返回一个(K,U)的数据集，使用指定的组合函数以及一个中立的"零"值。允许一个与输入值类型不同聚合值类型，用于避免没必要的分配。与<code>groupByKey</code>相似，reduce任务的数量可以通过可选的第二个参数来指定。</td>
</tr>
<tr>
  <td> <b>sortByKey</b>([<i>ascending</i>], [<i>numTasks</i>]) <a name="SortByLink"></a> </td>
  <td> 在一个(K, V)对(K是可排序的)的数据集上调用时，返回一个根据key升/降序排列的(K, V)数据集，通过<code>ascending</code>参数来指定升序还是降序。</td>
</tr>
<tr>
  <td> <b>join</b>(<i>otherDataset</i>, [<i>numTasks</i>]) <a name="JoinLink"></a> </td>
  <td> 在一个(K, V)数据集和一个(K, W)数据集上调用时，返回一个以K为key的所有V, W元素组成的tuple为值的(K, (V, W))型数据集。外连接可以通过
	<code>leftOuterJoin</code>, <code>rightOuterJoin</code>以及<code>fullOuterJoin</code>来实现。
  </td>
</tr>
<tr>
  <td> <b>cogroup</b>(<i>otherDataset</i>, [<i>numTasks</i>]) <a name="CogroupLink"></a> </td>
  <td> 在一个(K, V)数据集和一个(K, W)数据集上调用时，返回一个(K, (Iterable&lt;V&gt;, Iterable&lt;W&gt;))类型的数据集。该操作也被叫做<code>groupWith</code>。 </td>
</tr>
<tr>
  <td> <b>cartesian</b>(<i>otherDataset</i>) </td>
  <td> 在一个T类型数据集和一个U类型数据集上调用时，返回一个(T, U)类型的数据集(包含所有T和U元素)。(笛卡尔积)</td>
</tr>
<tr>
  <td> <b>pipe</b>(<i>command</i>, <i>[envVars]</i>) </td>
  <td> 通过一个shell命令的管道来传递RDD的每个分区，可以是perl或bash命令。 RDD元素以字符串的行是被写入到进程的标准输入stdin然后按行输出至标准输出stdout中。 </td>
</tr>
<tr>
  <td> <b>coalesce</b>(<i>numPartitions</i>) <a name="CoalesceLink"></a> </td>
  <td> 将RDD的分区数目减少至numPartitions。适用于需要在过滤完一个大的数据集后更有效率的运行。</td>
</tr>
<tr>
  <td> <b>repartition</b>(<i>numPartitions</i>) </td>
  <td> 对RDD内数据随机的重新进行shuffle以创建更多或更少的分区，然后将数据重新平均分配至这些分区。该操作总会将整个网络内的数据进行shuffle。 <a name="RepartitionLink"></a></td>
</tr>
<tr>
  <td> <b>repartitionAndSortWithinPartitions</b>(<i>partitioner</i>) <a name="Repartition2Link"></a></td>
  <td> 针对指定的partitioner来重新分配RDD的分区，然后在每个重新分配的分区内将数据按key重新排序。该操作要比直接调用<code>repartition</code>后再在每个分区内进行排序操作要有效率的多，因为它将排序操作下调进了shuffle操作中。
  </td>
</tr>
</table>

### Actions

下表列出了一些Spark里常用的Action。详情请参看RDD API文档
([Scala](api/scala/index.html#org.apache.spark.rdd.RDD),
 [Java](api/java/index.html?org/apache/spark/api/java/JavaRDD.html),
 [Python](api/python/pyspark.html#pyspark.RDD),
 [R](api/R/index.html))
 
以及键值对RDD函数文档
([Scala](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions),
 [Java](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html))

<table class="table">
<tr><th>Action</th><th>Meaning</th></tr>
<tr>
  <td> <b>reduce</b>(<i>func</i>) </td>
  <td> Aggregate the elements of the dataset using a function <i>func</i> (which takes two arguments and returns one). The function should be commutative and associative so that it can be computed correctly in parallel. </td>
</tr>
<tr>
  <td> <b>collect</b>() </td>
  <td> Return all the elements of the dataset as an array at the driver program. This is usually useful after a filter or other operation that returns a sufficiently small subset of the data. </td>
</tr>
<tr>
  <td> <b>count</b>() </td>
  <td> Return the number of elements in the dataset. </td>
</tr>
<tr>
  <td> <b>first</b>() </td>
  <td> Return the first element of the dataset (similar to take(1)). </td>
</tr>
<tr>
  <td> <b>take</b>(<i>n</i>) </td>
  <td> Return an array with the first <i>n</i> elements of the dataset. </td>
</tr>
<tr>
  <td> <b>takeSample</b>(<i>withReplacement</i>, <i>num</i>, [<i>seed</i>]) </td>
  <td> Return an array with a random sample of <i>num</i> elements of the dataset, with or without replacement, optionally pre-specifying a random number generator seed.</td>
</tr>
<tr>
  <td> <b>takeOrdered</b>(<i>n</i>, <i>[ordering]</i>) </td>
  <td> Return the first <i>n</i> elements of the RDD using either their natural order or a custom comparator. </td>
</tr>
<tr>
  <td> <b>saveAsTextFile</b>(<i>path</i>) </td>
  <td> Write the elements of the dataset as a text file (or set of text files) in a given directory in the local filesystem, HDFS or any other Hadoop-supported file system. Spark will call toString on each element to convert it to a line of text in the file. </td>
</tr>
<tr>
  <td> <b>saveAsSequenceFile</b>(<i>path</i>) <br /> (Java and Scala) </td>
  <td> Write the elements of the dataset as a Hadoop SequenceFile in a given path in the local filesystem, HDFS or any other Hadoop-supported file system. This is available on RDDs of key-value pairs that implement Hadoop's Writable interface. In Scala, it is also
   available on types that are implicitly convertible to Writable (Spark includes conversions for basic types like Int, Double, String, etc). </td>
</tr>
<tr>
  <td> <b>saveAsObjectFile</b>(<i>path</i>) <br /> (Java and Scala) </td>
  <td> Write the elements of the dataset in a simple format using Java serialization, which can then be loaded using
    <code>SparkContext.objectFile()</code>. </td>
</tr>
<tr>
  <td> <b>countByKey</b>() <a name="CountByLink"></a> </td>
  <td> Only available on RDDs of type (K, V). Returns a hashmap of (K, Int) pairs with the count of each key. </td>
</tr>
<tr>
  <td> <b>foreach</b>(<i>func</i>) </td>
  <td> Run a function <i>func</i> on each element of the dataset. This is usually done for side effects such as updating an <a href="#AccumLink">Accumulator</a> or interacting with external storage systems. 
  <br /><b>Note</b>: modifying variables other than Accumulators outside of the <code>foreach()</code> may result in undefined behavior. See <a href="#ClosuresLink">Understanding closures </a> for more details.</td>
</tr>
</table>

### Shuffle operations

Certain operations within Spark trigger an event known as the shuffle. The shuffle is Spark's
mechanism for re-distributing data so that it's grouped differently across partitions. This typically
involves copying data across executors and machines, making the shuffle a complex and
costly operation.

#### Background

To understand what happens during the shuffle we can consider the example of the
[`reduceByKey`](#ReduceByLink) operation. The `reduceByKey` operation generates a new RDD where all
values for a single key are combined into a tuple - the key and the result of executing a reduce
function against all values associated with that key. The challenge is that not all values for a
single key necessarily reside on the same partition, or even the same machine, but they must be
co-located to compute the result.

In Spark, data is generally not distributed across partitions to be in the necessary place for a
specific operation. During computations, a single task will operate on a single partition - thus, to
organize all the data for a single `reduceByKey` reduce task to execute, Spark needs to perform an
all-to-all operation. It must read from all partitions to find all the values for all keys, 
and then bring together values across partitions to compute the final result for each key - 
this is called the **shuffle**.

Although the set of elements in each partition of newly shuffled data will be deterministic, and so
is the ordering of partitions themselves, the ordering of these elements is not. If one desires predictably 
ordered data following shuffle then it's possible to use: 

* `mapPartitions` to sort each partition using, for example, `.sorted`
* `repartitionAndSortWithinPartitions` to efficiently sort partitions while simultaneously repartitioning
* `sortBy` to make a globally ordered RDD

Operations which can cause a shuffle include **repartition** operations like
[`repartition`](#RepartitionLink) and [`coalesce`](#CoalesceLink), **'ByKey** operations
(except for counting) like [`groupByKey`](#GroupByLink) and [`reduceByKey`](#ReduceByLink), and
**join** operations like [`cogroup`](#CogroupLink) and [`join`](#JoinLink).

#### Performance Impact
The **Shuffle** is an expensive operation since it involves disk I/O, data serialization, and
network I/O. To organize data for the shuffle, Spark generates sets of tasks - *map* tasks to
organize the data, and a set of *reduce* tasks to aggregate it. This nomenclature comes from
MapReduce and does not directly relate to Spark's `map` and `reduce` operations.

Internally, results from individual map tasks are kept in memory until they can't fit. Then, these 
are sorted based on the target partition and written to a single file. On the reduce side, tasks 
read the relevant sorted blocks.
        
Certain shuffle operations can consume significant amounts of heap memory since they employ 
in-memory data structures to organize records before or after transferring them. Specifically, 
`reduceByKey` and `aggregateByKey` create these structures on the map side, and `'ByKey` operations 
generate these on the reduce side. When data does not fit in memory Spark will spill these tables 
to disk, incurring the additional overhead of disk I/O and increased garbage collection.

Shuffle also generates a large number of intermediate files on disk. As of Spark 1.3, these files
are preserved until the corresponding RDDs are no longer used and are garbage collected. 
This is done so the shuffle files don't need to be re-created if the lineage is re-computed. 
Garbage collection may happen only after a long period time, if the application retains references 
to these RDDs or if GC does not kick in frequently. This means that long-running Spark jobs may 
consume a large amount of disk space. The temporary storage directory is specified by the
`spark.local.dir` configuration parameter when configuring the Spark context.

Shuffle behavior can be tuned by adjusting a variety of configuration parameters. See the
'Shuffle Behavior' section within the [Spark Configuration Guide](configuration.html). 

## RDD Persistence

One of the most important capabilities in Spark is *persisting* (or *caching*) a dataset in memory
across operations. When you persist an RDD, each node stores any partitions of it that it computes in
memory and reuses them in other actions on that dataset (or datasets derived from it). This allows
future actions to be much faster (often by more than 10x). Caching is a key tool for
iterative algorithms and fast interactive use.

You can mark an RDD to be persisted using the `persist()` or `cache()` methods on it. The first time
it is computed in an action, it will be kept in memory on the nodes. Spark's cache is fault-tolerant --
if any partition of an RDD is lost, it will automatically be recomputed using the transformations
that originally created it.

In addition, each persisted RDD can be stored using a different *storage level*, allowing you, for example,
to persist the dataset on disk, persist it in memory but as serialized Java objects (to save space),
replicate it across nodes, or store it off-heap in [Tachyon](http://tachyon-project.org/).
These levels are set by passing a
`StorageLevel` object ([Scala](api/scala/index.html#org.apache.spark.storage.StorageLevel),
[Java](api/java/index.html?org/apache/spark/storage/StorageLevel.html),
[Python](api/python/pyspark.html#pyspark.StorageLevel))
to `persist()`. The `cache()` method is a shorthand for using the default storage level,
which is `StorageLevel.MEMORY_ONLY` (store deserialized objects in memory). The full set of
storage levels is:

<table class="table">
<tr><th style="width:23%">Storage Level</th><th>Meaning</th></tr>
<tr>
  <td> MEMORY_ONLY </td>
  <td> Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, some partitions will
    not be cached and will be recomputed on the fly each time they're needed. This is the default level. </td>
</tr>
<tr>
  <td> MEMORY_AND_DISK </td>
  <td> Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the
    partitions that don't fit on disk, and read them from there when they're needed. </td>
</tr>
<tr>
  <td> MEMORY_ONLY_SER </td>
  <td> Store RDD as <i>serialized</i> Java objects (one byte array per partition).
    This is generally more space-efficient than deserialized objects, especially when using a
    <a href="tuning.html">fast serializer</a>, but more CPU-intensive to read.
  </td>
</tr>
<tr>
  <td> MEMORY_AND_DISK_SER </td>
  <td> Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of
    recomputing them on the fly each time they're needed. </td>
</tr>
<tr>
  <td> DISK_ONLY </td>
  <td> Store the RDD partitions only on disk. </td>
</tr>
<tr>
  <td> MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc.  </td>
  <td> Same as the levels above, but replicate each partition on two cluster nodes. </td>
</tr>
<tr>
  <td> OFF_HEAP (experimental) </td>
  <td> Store RDD in serialized format in <a href="http://tachyon-project.org">Tachyon</a>.
    Compared to MEMORY_ONLY_SER, OFF_HEAP reduces garbage collection overhead and allows executors
    to be smaller and to share a pool of memory, making it attractive in environments with
    large heaps or multiple concurrent applications. Furthermore, as the RDDs reside in Tachyon,
    the crash of an executor does not lead to losing the in-memory cache. In this mode, the memory
    in Tachyon is discardable. Thus, Tachyon does not attempt to reconstruct a block that it evicts
    from memory. If you plan to use Tachyon as the off heap store, Spark is compatible with Tachyon
    out-of-the-box. Please refer to this <a href="http://tachyon-project.org/master/Running-Spark-on-Tachyon.html">page</a>
    for the suggested version pairings.
  </td>
</tr>
</table>

**Note:** *In Python, stored objects will always be serialized with the [Pickle](https://docs.python.org/2/library/pickle.html) library, so it does not matter whether you choose a serialized level.*

Spark also automatically persists some intermediate data in shuffle operations (e.g. `reduceByKey`), even without users calling `persist`. This is done to avoid recomputing the entire input if a node fails during the shuffle. We still recommend users call `persist` on the resulting RDD if they plan to reuse it.

### Which Storage Level to Choose?

Spark's storage levels are meant to provide different trade-offs between memory usage and CPU
efficiency. We recommend going through the following process to select one:

* If your RDDs fit comfortably with the default storage level (`MEMORY_ONLY`), leave them that way.
  This is the most CPU-efficient option, allowing operations on the RDDs to run as fast as possible.

* If not, try using `MEMORY_ONLY_SER` and [selecting a fast serialization library](tuning.html) to
make the objects much more space-efficient, but still reasonably fast to access. 

* Don't spill to disk unless the functions that computed your datasets are expensive, or they filter
a large amount of the data. Otherwise, recomputing a partition may be as fast as reading it from
disk.

* Use the replicated storage levels if you want fast fault recovery (e.g. if using Spark to serve
requests from a web application). *All* the storage levels provide full fault tolerance by
recomputing lost data, but the replicated ones let you continue running tasks on the RDD without
waiting to recompute a lost partition.

* In environments with high amounts of memory or multiple applications, the experimental `OFF_HEAP`
mode has several advantages:
   * It allows multiple executors to share the same pool of memory in Tachyon.
   * It significantly reduces garbage collection costs.
   * Cached data is not lost if individual executors crash.

### Removing Data

Spark automatically monitors cache usage on each node and drops out old data partitions in a
least-recently-used (LRU) fashion. If you would like to manually remove an RDD instead of waiting for
it to fall out of the cache, use the `RDD.unpersist()` method.

# Shared Variables

Normally, when a function passed to a Spark operation (such as `map` or `reduce`) is executed on a
remote cluster node, it works on separate copies of all the variables used in the function. These
variables are copied to each machine, and no updates to the variables on the remote machine are
propagated back to the driver program. Supporting general, read-write shared variables across tasks
would be inefficient. However, Spark does provide two limited types of *shared variables* for two
common usage patterns: broadcast variables and accumulators.

## Broadcast Variables

Broadcast variables allow the programmer to keep a read-only variable cached on each machine rather
than shipping a copy of it with tasks. They can be used, for example, to give every node a copy of a
large input dataset in an efficient manner. Spark also attempts to distribute broadcast variables
using efficient broadcast algorithms to reduce communication cost.

Spark actions are executed through a set of stages, separated by distributed "shuffle" operations.
Spark automatically broadcasts the common data needed by tasks within each stage. The data
broadcasted this way is cached in serialized form and deserialized before running each task. This
means that explicitly creating broadcast variables is only useful when tasks across multiple stages
need the same data or when caching the data in deserialized form is important.

Broadcast variables are created from a variable `v` by calling `SparkContext.broadcast(v)`. The
broadcast variable is a wrapper around `v`, and its value can be accessed by calling the `value`
method. The code below shows this:

<div class="codetabs">

<div data-lang="scala"  markdown="1">

{% highlight scala %}
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

{% highlight java %}
Broadcast<int[]> broadcastVar = sc.broadcast(new int[] {1, 2, 3});

broadcastVar.value();
// returns [1, 2, 3]
{% endhighlight %}

</div>

<div data-lang="python"  markdown="1">

{% highlight python %}
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
{% endhighlight %}

</div>

</div>

After the broadcast variable is created, it should be used instead of the value `v` in any functions
run on the cluster so that `v` is not shipped to the nodes more than once. In addition, the object
`v` should not be modified after it is broadcast in order to ensure that all nodes get the same
value of the broadcast variable (e.g. if the variable is shipped to a new node later).

## Accumulators <a name="AccumLink"></a>

Accumulators are variables that are only "added" to through an associative operation and can
therefore be efficiently supported in parallel. They can be used to implement counters (as in
MapReduce) or sums. Spark natively supports accumulators of numeric types, and programmers
can add support for new types. If accumulators are created with a name, they will be
displayed in Spark's UI. This can be useful for understanding the progress of 
running stages (NOTE: this is not yet supported in Python).

An accumulator is created from an initial value `v` by calling `SparkContext.accumulator(v)`. Tasks
running on the cluster can then add to it using the `add` method or the `+=` operator (in Scala and Python).
However, they cannot read its value.
Only the driver program can read the accumulator's value, using its `value` method.

The code below shows an accumulator being used to add up the elements of an array:

<div class="codetabs">

<div data-lang="scala"  markdown="1">

{% highlight scala %}
scala> val accum = sc.accumulator(0, "My Accumulator")
accum: spark.Accumulator[Int] = 0

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum += x)
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Int = 10
{% endhighlight %}

While this code used the built-in support for accumulators of type Int, programmers can also
create their own types by subclassing [AccumulatorParam](api/scala/index.html#org.apache.spark.AccumulatorParam).
The AccumulatorParam interface has two methods: `zero` for providing a "zero value" for your data
type, and `addInPlace` for adding two values together. For example, supposing we had a `Vector` class
representing mathematical vectors, we could write:

{% highlight scala %}
object VectorAccumulatorParam extends AccumulatorParam[Vector] {
  def zero(initialValue: Vector): Vector = {
    Vector.zeros(initialValue.size)
  }
  def addInPlace(v1: Vector, v2: Vector): Vector = {
    v1 += v2
  }
}

// Then, create an Accumulator of this type:
val vecAccum = sc.accumulator(new Vector(...))(VectorAccumulatorParam)
{% endhighlight %}

In Scala, Spark also supports the more general [Accumulable](api/scala/index.html#org.apache.spark.Accumulable)
interface to accumulate data where the resulting type is not the same as the elements added (e.g. build
a list by collecting together elements), and the `SparkContext.accumulableCollection` method for accumulating
common Scala collection types.

</div>

<div data-lang="java"  markdown="1">

{% highlight java %}
Accumulator<Integer> accum = sc.accumulator(0);

sc.parallelize(Arrays.asList(1, 2, 3, 4)).foreach(x -> accum.add(x));
// ...
// 10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

accum.value();
// returns 10
{% endhighlight %}

While this code used the built-in support for accumulators of type Integer, programmers can also
create their own types by subclassing [AccumulatorParam](api/java/index.html?org/apache/spark/AccumulatorParam.html).
The AccumulatorParam interface has two methods: `zero` for providing a "zero value" for your data
type, and `addInPlace` for adding two values together. For example, supposing we had a `Vector` class
representing mathematical vectors, we could write:

{% highlight java %}
class VectorAccumulatorParam implements AccumulatorParam<Vector> {
  public Vector zero(Vector initialValue) {
    return Vector.zeros(initialValue.size());
  }
  public Vector addInPlace(Vector v1, Vector v2) {
    v1.addInPlace(v2); return v1;
  }
}

// Then, create an Accumulator of this type:
Accumulator<Vector> vecAccum = sc.accumulator(new Vector(...), new VectorAccumulatorParam());
{% endhighlight %}

In Java, Spark also supports the more general [Accumulable](api/java/index.html?org/apache/spark/Accumulable.html)
interface to accumulate data where the resulting type is not the same as the elements added (e.g. build
a list by collecting together elements).

</div>

<div data-lang="python"  markdown="1">

{% highlight python %}
>>> accum = sc.accumulator(0)
Accumulator<id=0, value=0>

>>> sc.parallelize([1, 2, 3, 4]).foreach(lambda x: accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
10
{% endhighlight %}

While this code used the built-in support for accumulators of type Int, programmers can also
create their own types by subclassing [AccumulatorParam](api/python/pyspark.html#pyspark.AccumulatorParam).
The AccumulatorParam interface has two methods: `zero` for providing a "zero value" for your data
type, and `addInPlace` for adding two values together. For example, supposing we had a `Vector` class
representing mathematical vectors, we could write:

{% highlight python %}
class VectorAccumulatorParam(AccumulatorParam):
    def zero(self, initialValue):
        return Vector.zeros(initialValue.size)

    def addInPlace(self, v1, v2):
        v1 += v2
        return v1

# Then, create an Accumulator of this type:
vecAccum = sc.accumulator(Vector(...), VectorAccumulatorParam())
{% endhighlight %}

</div>

</div>

For accumulator updates performed inside <b>actions only</b>, Spark guarantees that each task's update to the accumulator 
will only be applied once, i.e. restarted tasks will not update the value. In transformations, users should be aware 
of that each task's update may be applied more than once if tasks or job stages are re-executed.

Accumulators do not change the lazy evaluation model of Spark. If they are being updated within an operation on an RDD, their value is only updated once that RDD is computed as part of an action. Consequently, accumulator updates are not guaranteed to be executed when made within a lazy transformation like `map()`. The below code fragment demonstrates this property:

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% highlight scala %}
val accum = sc.accumulator(0)
data.map { x => accum += x; f(x) }
// Here, accum is still 0 because no actions have caused the `map` to be computed.
{% endhighlight %}
</div>

<div data-lang="java"  markdown="1">
{% highlight java %}
Accumulator<Integer> accum = sc.accumulator(0);
data.map(x -> { accum.add(x); return f(x); });
// Here, accum is still 0 because no actions have caused the `map` to be computed.
{% endhighlight %}
</div>

<div data-lang="python"  markdown="1">
{% highlight python %}
accum = sc.accumulator(0)
def g(x):
  accum.add(x)
  return f(x)
data.map(g)
# Here, accum is still 0 because no actions have caused the `map` to be computed.
{% endhighlight %}
</div>

</div>

# Deploying to a Cluster

The [application submission guide](submitting-applications.html) describes how to submit applications to a cluster.
In short, once you package your application into a JAR (for Java/Scala) or a set of `.py` or `.zip` files (for Python),
the `bin/spark-submit` script lets you submit it to any supported cluster manager.

# Launching Spark jobs from Java / Scala

The [org.apache.spark.launcher](api/java/index.html?org/apache/spark/launcher/package-summary.html)
package provides classes for launching Spark jobs as child processes using a simple Java API.

# Unit Testing

Spark is friendly to unit testing with any popular unit test framework.
Simply create a `SparkContext` in your test with the master URL set to `local`, run your operations,
and then call `SparkContext.stop()` to tear it down.
Make sure you stop the context within a `finally` block or the test framework's `tearDown` method,
as Spark does not support two contexts running concurrently in the same program.

# Migrating from pre-1.0 Versions of Spark

<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark 1.0 freezes the API of Spark Core for the 1.X series, in that any API available today that is
not marked "experimental" or "developer API" will be supported in future versions.
The only change for Scala users is that the grouping operations, e.g. `groupByKey`, `cogroup` and `join`,
have changed from returning `(Key, Seq[Value])` pairs to `(Key, Iterable[Value])`.

</div>

<div data-lang="java"  markdown="1">

Spark 1.0 freezes the API of Spark Core for the 1.X series, in that any API available today that is
not marked "experimental" or "developer API" will be supported in future versions.
Several changes were made to the Java API:

* The Function classes in `org.apache.spark.api.java.function` became interfaces in 1.0, meaning that old
  code that `extends Function` should `implement Function` instead.
* New variants of the `map` transformations, like `mapToPair` and `mapToDouble`, were added to create RDDs
  of special data types.
* Grouping operations like `groupByKey`, `cogroup` and `join` have changed from returning 
  `(Key, List<Value>)` pairs to `(Key, Iterable<Value>)`.

</div>

<div data-lang="python"  markdown="1">

Spark 1.0 freezes the API of Spark Core for the 1.X series, in that any API available today that is
not marked "experimental" or "developer API" will be supported in future versions.
The only change for Python users is that the grouping operations, e.g. `groupByKey`, `cogroup` and `join`,
have changed from returning (key, list of values) pairs to (key, iterable of values).

</div>

</div>

Migration guides are also available for [Spark Streaming](streaming-programming-guide.html#migration-guide-from-091-or-below-to-1x),
[MLlib](mllib-guide.html#migration-guide) and [GraphX](graphx-programming-guide.html#migrating-from-spark-091).


# Where to Go from Here

You can see some [example Spark programs](http://spark.apache.org/examples.html) on the Spark website.
In addition, Spark includes several samples in the `examples` directory
([Scala]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples),
 [Java]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples),
 [Python]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python),
 [R]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/r)).
You can run Java and Scala examples by passing the class name to Spark's `bin/run-example` script; for instance:

    ./bin/run-example SparkPi

For Python examples, use `spark-submit` instead:

    ./bin/spark-submit examples/src/main/python/pi.py

For R examples, use `spark-submit` instead:

    ./bin/spark-submit examples/src/main/r/dataframe.R

For help on optimizing your programs, the [configuration](configuration.html) and
[tuning](tuning.html) guides provide information on best practices. They are especially important for
making sure that your data is stored in memory in an efficient format.
For help on deploying, the [cluster mode overview](cluster-overview.html) describes the components involved
in distributed operation and supported cluster managers.

Finally, full API documentation is available in
[Scala](api/scala/#org.apache.spark.package), [Java](api/java/), [Python](api/python/) and [R](api/R/).
