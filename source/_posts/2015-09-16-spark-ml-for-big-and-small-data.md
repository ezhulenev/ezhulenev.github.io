---
layout: post
title: "Optimizing Spark Machine Learning for Small Data"
date: 2015-09-16 09:04:25 -0500
comments: true
categories: [spark, scala, dataframe, machine learning]
keywords: spark, scala, dataframe, machine learning
---

You've all probably already know how awesome is Spark for doing Machine Learning on Big Data. However I'm pretty sure
no one told you how bad (slow) it can be on Small Data. 

As I mentioned in my [previous post](/blog/2015/09/09/audience-modeling-with-spark-ml-pipelines), we
extensively use Spark for doing machine learning and audience modeling. It turned out that in some cases, for example when
we are starting optimization for new client/campaign we simply don't have enough positive examples to construct big enough dataset, so that
using Spark would make sense.

<!-- more -->

### Spark ML from 10000 feet

Essentially every machine learning algorithm is a function minimization, where function value depends on some calculation using data in `RDD`.
For example logistic regression can calculate function value 1000 times before it will converge and find optimal parameters. It means that it will 
compute some `RDD` 1000 times. In case of `LogisticRegression` it's doing `RDD.treeAggregate` which is supper efficient, but still it's distributed 
computation.

Now imagine that all the data you have is 50000 rows, and you have for example 1000 partitions. It means that each partition has only 50 rows. And 
each `RDD.treeAggregate` on every iteration serializing closures, sending them to partitions and collecting result back. 
It's **HUGE OVERHEAD** and huge load on a driver.


### Throw Away Spark and use Python/R?

It's definitely an option, but we don't want to build multiple systems for data of different size. Spark ML pipelines are awesome abstraction,
and we want to use it for all machine learning jobs. Also we want to use the same algorithm, so results would be consistent if dataset size
just crossed the boundary between small and big data.

### Run LogisticRegression in 'Local Mode'

What if Spark could run the same machine learning algorithm, but instead of using `RDD` for storing input data, it would use `Arrays`?
It solves all the problems, you get consistent model, computed 10-20x faster because it doesn't need distributed computations.

That's exactly approach I used in [Spark Ext](https://github.com/collectivemedia/spark-ext), it's called [LocalLogisticRegression](https://github.com/collectivemedia/spark-ext/blob/master/sparkext-mllib/src/main/scala/org/apache/spark/ml/classification/LocalLogisticRegression.scala).
It's mostly copy-pasta from Spark `LogisticRegression`, but when input data frame has only single partition, it's running
function optimization on one of the executors using `mapPartition` function, essentially using Spark as distributed executor service.

This approach is much better than collecting data to driver, because you are not limited by driver computational resources.

When `DataFrame` has more than 1 partition it just falls back to default distributed logistic regression.

Code for new `train` method looks like this:

{% coderay lang:groovy %}
def trainLocal(
      instances: Array[(Double, Vector)]
    ): (LogisticRegressionModel, Array[Double]) = ...

def train(dataset: DataFrame): LogisticRegressionModel = {

  if (dataset.rdd.partitions.length == 1) {
    log.info(s"Build LogisticRegression in local mode")

    val (model, objectiveHistory) = extractLabeledPoints(dataset).map {
      case LabeledPoint(label: Double, features: Vector) => (label, features)
    }.mapPartitions { instances =>
      Seq(trainLocal(instances.toArray)).toIterator
    }.first()

    val logRegSummary = new BinaryLogisticRegressionTrainingSummary(
      model.transform(dataset),
      probabilityCol,
      labelCol,
      objectiveHistory)
    model.setSummary(logRegSummary)

  } else {
    log.info(s"Fallback to distributed LogisticRegression")

    val that = classOf[LogisticRegression].getConstructor(classOf[String]).newInstance(uid)
    val logisticRegression = copyValues(that)
    // Scala Reflection magic to call protected train method
    ...
    logisticRegression.train(dataset)
  }
}      
{% endcoderay %}

If input dataset size is less than 100000 rows, it will be placed inside single partition, and regression model will be trained in local mode.
   
{% coderay lang:groovy %}
val base: DataFrame = ...
val datasetPartitionSize = 100000

// Compute optimal partitions size based on base join
val baseSize = base.count()
val numPartitions = (baseSize.toDouble / datasetPartitionSize).ceil.toInt
log.debug(s"Response base size: $baseSize")
log.debug(s"Repartition dataset using $numPartitions partitions")
{% endcoderay %}

## Results

With a little ingenuity (and copy paste) Spark became perfect tool for machine learning both on Small and Big Data. Most awesome thing is that this
new `LocalLogisticRegression` can be used as drop in replacement in Spark ML pipelines, producing exactly the same `LogisticRegressionModel` at the end.

It might be interesting idea to use this approach in Spark itself, because in this case it would be possible to do it
without doing so many code duplication. I'd love to see if anyone else had the same problem, and how solved it.

> More cool Spark things in [Github](https://github.com/collectivemedia/spark-ext/).
