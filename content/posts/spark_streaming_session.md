---
title: 怎样利用Spark Streaming和Hadoop实现近实时的会话连接
date: 2015-1-6 10:20
category:
- big-data
- spark
tags:
- big-data
- spark
---


这个 Spark Streaming 样例是一个可持久化到Hadoop近实时会话的很好的例子。

[Spark Streaming](https://spark.apache.org/streaming) 是Apache Spark 中最有趣的组件之一。你用Spark Streaming可以创建数据管道来用批量加载数据一样的API处理流式数据。此外，Spark Steaming的“micro-batching”方式提供相当好的弹性来应对一些原因造成的任务失败。

在这篇文章中，我将通过网站的事件近实时回话的例子演示使你熟悉一些常见的和高级的Spark Streaming功能，然后加载活动有关的统计数据到Apache HBase，用不喜欢的BI用具来绘图分析。 ([Sessionization](http://en.wikipedia.org/wiki/Sessionization)指的是捕获的单一访问者的网站会话时间范围内所有点击流活动。)你可以在[这里](https://github.com/tmalaska/SparkStreaming.Sessionization)找到了这个演示的代码。

像这样的系统对于了解访问者的行为（无论是人还是机器）是超级有用的。通过一些额外的工作它也可以被设计成windowing模式来以异步方式检测可能的欺诈。

<!-- more -->
Spark Streaming 代码

我们的例子中的main class是：

`com.cloudera.sa.example.sparkstreaming.sessionization.SessionizeData`

让我们来看看这段代码段（忽略1-59行，其中包含imports 和其他无聊的东西）。

60到112行：设置Spark Streaming 这些行是非常基本的，用来设置的Spark Streaming，同时可以选择从HDFS或socket接收数据流。如果你在Spark Streaming方面是一个新手，我已经添加了一些详细的注释帮助理解代码。 （我不打算在这里详谈，因为仍然在样例代码里。）

```scala
//This is just creating a Spark Config object.  I don't do much here but

//add the app name.  There are tons of options to put into the Spark config,

//but none are needed for this simple example.
    val sparkConf = new SparkConf().
      setAppName("SessionizeData " + args(0)).
      set("spark.cleaner.ttl", "120000")

//These two lines will get us out SparkContext and our StreamingContext.

//These objects have all the root functionality we need to get started.
    val sc = new SparkContext(sparkConf)
    val ssc = new StreamingContext(sc, Seconds(10))

//Here are are loading our HBase Configuration object.  This will have

//all the information needed to connect to our HBase cluster.

//There is nothing different here from when you normally interact with HBase.
    val conf = HBaseConfiguration.create();
    conf.addResource(new Path("/etc/hbase/conf/core-site.xml"));
    conf.addResource(new Path("/etc/hbase/conf/hbase-site.xml"));

//This is a HBaseContext object.  This is a nice abstraction that will hide

//any complex HBase stuff from us so we can focus on our business case

//HBaseContext is from the SparkOnHBase project which can be found at

// https://github.com/tmalaska/SparkOnHBase
    val hbaseContext = new HBaseContext(sc, conf);

//This is create a reference to our root DStream.  DStreams are like RDDs but

//with the context of being in micro batch world.  I set this to null now

//because I later give the option of populating this data from HDFS or from

//a socket.  There is no reason this could not also be populated by Kafka,

//Flume, MQ system, or anything else.  I just focused on these because

//there are the easiest to set up.
    var lines: DStream[String] = null

//Options for data load.  Will be adding Kafka and Flume at some point
    if (args(0).equals("socket")) {
      val host = args(FIXED_ARGS);
      val port = args(FIXED_ARGS + 1);

      println("host:" + host)
      println("port:" + Integer.parseInt(port))

//Simple example of how you set up a receiver from a Socket Stream
      lines = ssc.socketTextStream(host, port.toInt)
    } else if (args(0).equals("newFile")) {

      val directory = args(FIXED_ARGS)
      println("directory:" + directory)

//Simple example of how you set up a receiver from a HDFS folder
      lines = ssc.fileStream[LongWritable, Text, TextInputFormat](directory, (t: Path) =&gt; true, true).map(_._2.toString)
    } else {
      throw new RuntimeException("bad input type")
    }
```

114到124行: 字符串解析 这里是Spark Streaming的开始的地方. 请看下面四行：:

```scala
val ipKeyLines = lines.map[(String, (Long, Long, String))](eventRecord =&gt; {

//Get the time and ip address out of the original event
      val time = dateFormat.parse(
        eventRecord.substring(eventRecord.indexOf('[') + 1, eventRecord.indexOf(']'))).
        getTime()
      val ipAddress = eventRecord.substring(0, eventRecord.indexOf(' '))

//We are return the time twice because we will use the first at the start time

//and the second as the end time
      (ipAddress, (time, time, eventRecord))
    })
```

上面第一命令是在DSTREAM对象“lines”上进行了map函数和，解析原始事件来分离出的IP地址，时间戳和事件的body。对于那些Spark Streaming的新手，一个DSTREAM保存着要处理的一批记录。这些记录由以前所定义的receiver对象填充，并且此map函数在这个micro-batch内产生另一个DSTREAM存储变换后的记录来进行额外的处理。

![image](https://dn-mtunique.qbox.me/sessionization-f11.png)

当看像上面的Spark Streaming示意图时，有一些事情要注意：:

*   每个micro-batch在到达构建StreamingContext时设定的那一秒时被销毁
*   Receiver总是用被下一个micro-batch中的RDDS填充
*   之前micro batch中老的RDDs将被清理丢弃

126到135行：产生Sessions 现在，我们有从网络日志中获得的IP地址和时间，是时候建立sessions了。下面的代码是通过micro-batch内的第一聚集事件建立session，然后在DSTREAM中reduce这些会话。

```scala
val latestSessionInfo = ipKeyLines.
      map[(String, (Long, Long, Long))](a =&gt; {

//transform to (ipAddress, (time, time, counter))
        (a._1, (a._2._1, a._2._2, 1))
      }).
      reduceByKey((a, b) =&gt; {

//transform to (ipAddress, (lowestStartTime, MaxFinishTime, sumOfCounter))
        (Math.min(a._1, b._1), Math.max(a._2, b._2), a._3 + b._3)
      }).
      updateStateByKey(updateStatbyOfSessions)
```

这里有一个关于records如何在micro-batch中被reduce的例子： ![image](https://dn-mtunique.qbox.me/sessionization-table.png)

在会话范围内的 micro-batch 内加入，我们可以用超酷的updateStateByKey功能（做join/reduce-like操作）下图说明了就DStreams而言，随着时间变化这个处理过程是怎样的。

![image](https://dn-mtunique.qbox.me/sessionization-f2.png)

现在，让我们深入到updateStatbyOfSessions函数，它被定义在文件的底部。此代码（注意详细注释）含有大量的魔法，使sessionization发生在micro-batch的连续模式中。

```scala
/**

* This function will be called for to union of keys in the Reduce DStream

* with the active sessions from the last micro batch with the ipAddress

* being the key

*

* To goal is that this produces a stateful RDD that has all the active

* sessions.  So we add new sessions and remove sessions that have timed

* out and extend sessions that are still going

*/
  def updateStatbyOfSessions(

//(sessionStartTime, sessionFinishTime, countOfEvents)
      a: Seq[(Long, Long, Long)],

//(sessionStartTime, sessionFinishTime, countOfEvents, isNewSession)
      b: Option[(Long, Long, Long, Boolean)]
    ): Option[(Long, Long, Long, Boolean)] = {

//This function will return a Optional value.

//If we want to delete the value we can return a optional "None".

//This value contains four parts

//(startTime, endTime, countOfEvents, isNewSession)
    var result: Option[(Long, Long, Long, Boolean)] = null

// These if statements are saying if we didn't get a new event for

//this session's ip address for longer then the session

//timeout + the batch time then it is safe to remove this key value

//from the future Stateful DStream
    if (a.size == 0) {
      if (System.currentTimeMillis() - b.get._2 &gt; SESSION_TIMEOUT + 11000) {
        result = None
      } else {
        if (b.get._4 == false) {
          result = b
        } else {
          result = Some((b.get._1, b.get._2, b.get._3, false))
        }
      }
    }

//Now because we used the reduce function before this function we are

//only ever going to get at most one event in the Sequence.
    a.foreach(c =&gt; {
      if (b.isEmpty) {

//If there was no value in the Stateful DStream then just add it

//new, with a true for being a new session
        result = Some((c._1, c._2, c._3, true))
      } else {
        if (c._1 - b.get._2 &lt; SESSION_TIMEOUT) {

//If the session from the stateful DStream has not timed out

//then extend the session
          result = Some((
              Math.min(c._1, b.get._1),
//newStartTime
              Math.max(c._2, b.get._2),
//newFinishTime
              b.get._3 + c._3,
//newSumOfEvents
              false
//This is not a new session
            ))
        } else {

//Otherwise remove the old session with a new one
          result = Some((
              c._1,
//newStartTime
              c._2,
//newFinishTime
              b.get._3,
//newSumOfEvents
              true
//new session
            ))
        }
      }
    })
    result
  }
}
```

在这段代码做了很多事，而且通过很多方式，这是整个工作中最复杂的部分。总之，它跟踪活动的会话，所以你知道你是继续现有的会话还是启动一个新的。

126到207行：计数和HBase 这部分做了大多数计数工作。在这里有很多是重复的，让我们只看一个count的例子，然后一步步地我们把生成的同一个记录counts存储在HBase中。

```scala
val onlyActiveSessions = latestSessionInfo.filter(t =&gt; System.currentTimeMillis() - t._2._2 &lt; SESSION_TIMEOUT)
…
val newSessionCount = onlyActiveSessions.filter(t =&gt; {

//is the session newer then that last micro batch

//and is the boolean saying this is a new session true
        (System.currentTimeMillis() - t._2._2 &gt; 11000 &amp;&amp; t._2._4)
      }).
      count.
      map[HashMap[String, Long]](t =&gt; HashMap((NEW_SESSION_COUNTS, t)))
```

总之，上面的代码是过滤除了活动的会话其他所有会话，对他们进行计数，并把该最终计记录到一个的HashMap实例中。它使用HashMap作为容器，所以在所有的count做完后，我们可以调用下面的reduce函数把他们都到一个单一的记录。 （我敢肯定有更好的方法来实现这一点，但这种方法工作得很好。）

接下来，下面的代码处理所有的那些HashMap，并把他们所有的值在一个HashMap中。

```scala
val allCounts = newSessionCount.
      union(totalSessionCount).
      union(totals).
      union(totalEventsCount).
      union(deadSessionsCount).
      union(totalSessionEventCount).
      reduce((a, b) =&gt; b ++ a)
```

用HBaseContext来使Spark Streaming与HBase交互超级简单。所有你需要做的就是用HashMap和函数将其转换为一个put对象提供给DSTREAM。

```scala
hbaseContext.streamBulkPut[HashMap[String, Long]](
      allCounts,
//The input RDD
      hTableName,
//The name of the table we want to put too
      (t) =&gt; {

//Here we are converting our input record into a put

//The rowKey is C for Count and a backward counting time so the newest

//count show up first in HBase's sorted order
        val put = new Put(Bytes.toBytes("C." + (Long.MaxValue - System.currentTimeMillis())))

//We are iterating through the HashMap to make all the columns with their counts
        t.foreach(kv =&gt; put.add(Bytes.toBytes(hFamily), Bytes.toBytes(kv._1), Bytes.toBytes(kv._2.toString)))
        put
      },
      false)
```

现在，HBase的这些信息可以用Apache Hive table包起来，然后通过你喜欢的BI工具执行一个查询来获取像下面这样的图，它每次micro-batch会刷新。 ![image](https://dn-mtunique.qbox.me/sessionization-f3.png)

209到215行：写入HDFS 最后的任务是把拥有事件数据的活动会话信息加入，然后把事件以会话的开始时间来持久化到HDFS。
````scala
//Persist to HDFS
ipKeyLines.join(onlyActiveSessions).
  map(t =&gt; {

//Session root start time | Event message
    dateFormat.format(new Date(t._2._2._1)) + "t" + t._2._1._3
  }).
  saveAsTextFiles(outputDir + "/session", "txt")
```

结论

我希望你跳出这个例子 来走像了很多工作，感觉与代码只是一点点做，因为它是。想象一下你还可以用这种模式和Spark Streaming与HBase HDFS很容易交互的这种能力做什么东西。

[原文](http://blog.cloudera.com/blog/2014/11/how-to-do-near-real-time-sessionization-with-spark-streaming-and-apache-hadoop/)
