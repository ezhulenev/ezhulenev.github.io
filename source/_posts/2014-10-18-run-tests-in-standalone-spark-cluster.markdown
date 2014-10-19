---
layout: post
title: "Running Spark tests in standalone cluster"
date: 2014-10-18 21:01:15 -0400
comments: true
categories: [spark, scala, sbt]
keywords: Spark, Scala, sbt, Apache Spark, Apache Spark tutorial, Big Data Spark, How to make Spark Single Jar, Spark assembly, Spark fat jar, Spark sbt, Spark sbt assembly, Spark Scala, Spark uber jar
---

> The code for this application app can be found on [Github](https://github.com/ezhulenev/spark-testing)

### Running Spark Applications

To be able to run Spark jobs, Spark cluster needs to have all classes used by your application in it's classpath.
You can put manually all jar files required by your application to Spark nodes, but it's not cool.
Another solution is to manually set jar files that required to distribute to worker nodes
when you create SparkConf. One way to do it, is to package your application as a "fat-jar",
so you need to distribute only single jar.
Industry standard for packaging Spark application is [sbt-assembly](https://github.com/sbt/sbt-assembly) plugin,
and it's used by Spark itself.

### Testing Spark Applications

If you need to test your Spark application, easiest way is to create local Spark Context for each test, or maybe shared between all tests.
When Spark is running in local mode, it's running in the same JVM as your tests with same jar files in classpath.

If your tests requires data that doesn't fit into single node, obvious solution is to run them in standalone Spark cluster
with sufficient number of nodes. At this time everything becomes more difficult. Now you need to package you application with tests
in single jar file, and submit it to Spark cluster with each test.

<!-- more -->

### Example Application

To show how to run and test Spark applications I prepared very [simple application](https://github.com/ezhulenev/spark-testing).
It uses [Scala OpenBook](https://github.com/ezhulenev/scala-openbook)
library to parse [NYSE OpenBook](http://www.nyxdata.com/Data-Products/NYSE-OpenBook-History) messages (orders log from New York Stock Exchange),
distribute them to cluster as RDD, and count Buy and Sell orders by ticker.
Only purpose of this application is to have dependency on a library that for sure is not available on Spark nodes.

{% coderay lang:groovy %}
class OrdersFunctions(@transient sc: SparkContext, orders: Iterator[OpenBookMsg]) extends Serializable {

  private val ordersRDD = sc.parallelize(orders.toSeq)

  def countBuyOrders(): Map[String, Long] = countOrders(OrderFunctions.isBuySide)

  def countSellOrders(): Map[String, Long] = countOrders(OrderFunctions.isSellSide)

  private def countOrders(filter: OpenBookMsg => Boolean): Map[String, Long] =
    ordersRDD.filter(filter).
      map(order => (order.symbol, order)).
      countByKey().toMap

}
{% endcoderay %}

&nbsp;

### Assembly Main Application

Add sbt-assembly plugin in project/plugin.sbt

{% coderay lang:groovy %}
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.11.2")
{% endcoderay %}

Add assembly settings to build.sbt

{% coderay lang:groovy %}
// Merge strategy shared between app & test

val sharedMergeStrategy: (String => MergeStrategy) => String => MergeStrategy =
  old => {
    case x if x.startsWith("META-INF/ECLIPSEF.RSA") => MergeStrategy.last
    case x if x.startsWith("META-INF/mailcap") => MergeStrategy.last
    case x if x.endsWith("plugin.properties") => MergeStrategy.last
    case x => old(x)
  }

// Load Assembly Settings

assemblySettings

// Assembly App

mainClass in assembly := Some("com.github.ezhulenev.spark.RunSparkApp")

jarName in assembly := "spark-testing-example-app.jar"

mergeStrategy in assembly <<= (mergeStrategy in assembly)(sharedMergeStrategy)
{% endcoderay %}

Inside your application you need to create SparkConf and add current jar to it.

{% coderay lang:groovy %}
  new SparkConf().
      setMaster("spark://spark-host:7777").
      setJars(SparkContext.jarOfClass(this.getClass).toSeq).
      setAppName("SparkTestingExample")
{% endcoderay %}

After that you can use assembly command, and run assembled application in your Spark Cluster

{% coderay %}
> sbt assembly
> java -Dspark.master=spark://spark-host:7777 target/scala_2.10/spark-testing-example-app.jar
{% endcoderay %}

&nbsp;

### Assembly Tests

First step to run tests in standalone Spark Cluster is to package all main and test classes into single jar, that will be
transfered to each worker node before running tests. It's very similar to assemblying main app.

{% coderay lang:groovy %}
// Assembly Tests

Project.inConfig(Test)(assemblySettings)

jarName in (Test, assembly) := "spark-testing-example-tests.jar"

mergeStrategy in (Test, assembly) <<= (mergeStrategy in assembly)(sharedMergeStrategy)

test in (Test, assembly) := {} // disable tests in assembly
{% endcoderay %}

I wrote simple sbt plugin that has `test-assembly` task. First this task assemblies jar
file with test classes and all dependencies, then set it's location
to environment variable, and then starts tests.

{% coderay lang:groovy %}
object TestWithSparkPlugin extends sbt.Plugin {

  import TestWithSparkKeys._
  import AssemblyKeys._

  object TestWithSparkKeys {
    lazy val testAssembled        = TaskKey[Unit]("test-assembled", "Run tests with standalone Spark cluster")
    lazy val assembledTestsProp   = SettingKey[String]("assembled-tests-prop", "Environment variable name used to pass assembled jar name to test")
  }

  lazy val baseTestWithSparkSettings: Seq[sbt.Def.Setting[_]] = Seq(
    testAssembled        := TestWithSpark.testWithSparkTask.value,
    assembledTestsProp   := "ASSEMBLED_TESTS"
  )

  lazy val testWithSparkSettings: Seq[sbt.Def.Setting[_]] = baseTestWithSparkSettings

  object TestWithSpark {

    def assemblyTestsJarTask: Initialize[Task[File]] = Def.task {
      val assembled = (assembly in Test).value
      sys.props(assembledTestsProp.value) = assembled.getAbsolutePath
      assembled
    }

    private def runTests = Def.task {
      (test in Test).value
    }

    def testWithSparkTask: Initialize[Task[Unit]] = Def.sequentialTask {
      assemblyTestsJarTask.value
      runTests.value
    }
  }
}
{% endcoderay %}

All Spark tests should inherit `ConfiguredSparkFlatSpec` with configured Spark Context. If assembled tests jar file
is available, it's distributed to Spark worker nodes. If not, only local mode is supported.

{% coderay lang:groovy %}
trait ConfiguredSparkFlatSpec extends FlatSpec with BeforeAndAfterAll {
  private val log = LoggerFactory.getLogger(classOf[ConfiguredSparkFlatSpec])

  private val config = ConfigFactory.load()

  private lazy val sparkConf = {
    val master = config.getString("spark.master")

    log.info(s"Create spark context. Master: $master")
    val assembledTests = sys.props.get("ASSEMBLED_TESTS")

    val baseConf = new SparkConf().
      setMaster(master).
      setAppName("SparkTestingExample")

    assembledTests match {
      case None =>
        log.warn(s"Assembled tests jar not found. Standalone Spark mode is not supported")
        baseConf
      case Some(path) =>
        log.info(s"Add assembled tests to Spark Context from: $path")
        baseConf.setJars(path :: Nil)
    }
  }

  lazy val sc = new SparkContext(sparkConf)

  override protected def afterAll(): Unit = {
    super.afterAll()
    sc.stop()
  }
}
{% endcoderay %}

&nbsp;

### Running Tests

By default `spark.master` property is set to local[2]. So you can run tests in local mode. If you want run tests
in standalone Spark, you need to override `spark.master` with your master node.

If you'll try to run `test` command with standalone cluster it will fail with ClassNotFoundException

{% coderay %}
> sbt -Dspark.master=spark://spark-host:7777 test
>
> Create spark context. Master: spark://Eugenes-MacBook-Pro.local:7077
> Assembled tests jar not found. Standalone Spark mode is not supported
>
> [error] Failed tests:
> org.apache.spark.SparkException: Job aborted due to stage failure:
> Task 2 in stage 1.0 failed 4 times, most recent failure:
> Lost task 2.3 in stage 1.0 (TID 30, 192.168.0.11):
> java.lang.ClassNotFoundException: com.scalafi.openbook.OpenBookMsg
{% endcoderay %}

However `test-assembled` will be successfull

{% coderay %}
> sbt -Dspark.master=spark://spark-host:7777 test-assembled
>
> Create spark context. Master: spark://Eugenes-MacBook-Pro.local:7077
> Add assembled tests to Spark Context from: /Users/ezhulenev/spark-testing/target/scala-2.10/spark-testing-example-tests.jar
>
> [info] Run completed in 7 seconds, 587 milliseconds.
> [info] Total number of tests run: 2
> [info] Suites: completed 1, aborted 0
> [info] Tests: succeeded 2, failed 0, canceled 0, ignored 0, pending 0
> [info] All tests passed.
> [success] Total time: 37 s
{% endcoderay %}

&nbsp;

> The code for this application app can be found on [Github](https://github.com/ezhulenev/spark-testing)
