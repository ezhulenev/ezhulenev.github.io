---
layout: post
title: "Spark in Production: Lessons From Running Large Scale Machine Learning"
date: 2015-12-03 13:24:46 -0500
comments: true
categories: [spark, scala, dataframe, machine learning, scalaz-stream]
keywords: spark, scala, dataframe, machine learning, scalaz-stream
---

I wrote earlier about our approach for [machine learning with Spark](/blog/2015/09/09/audience-modeling-with-spark-ml-pipelines) 
at [Collective](http://collective.com), it was focused on transforming raw data into features that can be used for training a model.
At this post I want describe how to assemble multiple building blocks into production application, that efficiently uses
Spark cluster and can train/validate hundreds of models.

Training single model is relatively easy, and it's well covered in Spark documentation and multiple other blog posts. Training hundreds of 
models can become really tricky from engineering point of view. Spark has lot's of configuration parameters 
that can affect cluster performance and stability, and you can use some clever tricks to get higher cluster utilization.

<!-- more -->

### Scale Of The Problem

At Collective we are using Spark and machine learning for online advertising optimization, trying to decide which ads are relevant to 
which people, at which time and at which web site. 

Log data used for training models is huge, billions of rows. Number of users that we target is hundreds of millions. 
We have hundreds of clients with tens of different campaigns, with different optimization targets and restrictions.
  
These factors gives an idea of the scale of the problem.  

### Production Machine Learning Pipeline

Typical machine learning pipeline for one specific ad campaign looks like this:
  
  + *Prepare response dataset:* based on impression/activity logs define which users should be in positive set (did some actions that we are trying to optimize for, example could be signing up for test drive)
  + *Prepare train dataset:* extract predictors data from logs
  + *Feauturize:* given predictors data extract feature vectors
  + *Train model:* feed features and response data into Spark ML
  + *Prepare test dataset:* data that is going to be used for model performance evaluation
  + *Evaluate model:* for binary classification it can be computing ROC and AUC

Preparing datasets is usually contains of multiple join and filter conditions. Featurization and training built on top 
of [Spark ML Pipeline](https://databricks.com/blog/2015/01/07/ml-pipelines-a-new-high-level-api-for-mllib.html) API, and 
compose multiple transformers and estimators together. It's covered in [one of previous posts](/blog/2015/09/09/audience-modeling-with-spark-ml-pipelines).

Given different nature of each step they have different limiting factors. Preparing dataset is expensive operation and heavily 
uses shuffle. Model training time is usually dominated by network latency introduced by iterative function optimization algorithm.

Serial execution of these steps can't efficiently utilize all executor cores. Running multiple models in parallel in our 
case doesn't really help as well, multiple models reach *modeling* stage at almost the same time.

### Introducing Scalaz-Stream

I'll just put first paragraph from amazing [Introduction to scalaz-stream](https://gist.github.com/djspiewak/d93a9c4983f63721c41c) here:
  
> Every application ever written can be viewed as some sort of transformation on data. Data can come from different sources, such as a network or a file or user input or the Large Hadron Collider. It can come from many sources all at once to be merged and aggregated in interesting ways, and it can be produced into many different output sinks, such as a network or files or graphical user interfaces. You might produce your output all at once, as a big data dump at the end of the world (right before your program shuts down), or you might produce it more incrementally. Every application fits into this model.

We model machine learning pipeline as a `scalaz.stream.Process` - multistep transformation on data, and use `scalaz-stream` combinators to run it
with controlled concurrency and resource safety.

Simple domain model for campaign optimization can be defined like this:

{% coderay lang:groovy %}
case class OptimizedCampaign(id: Int, input: Input, target: Target)

case class ModelingError(error: String)

case class TrainedAndEvaluatedModel(...)
{% endcoderay %}

Input would be definition of campaign that needs to be optimized, and output would be model that was trained and evaluated or error if something went wrong.

Each of modeling steps described earlier can be encoded as `scalaz.stream.Channel` transformations:

{% coderay lang:groovy %}

import scalaz.stream._

val prepareResponse = 
  channel.lift[Task, OptimizedCampaign, ModelingError \/ ResponseDataset] {
    case opt =>
       // Expensive join/filter etc...
       val response: DataFrame = 
         dataset1
           .join(dataset2, ...)
           .filter(..)
           .select(...)
       ResponseDataset(response)
  }

val prepareTrainDataset = lift[ResponseDataset, TrainDataset] {
  case response =>
    // Another expensive joins that requires shuffle
    val train: DataFrame = 
      dataset1.join(dataset2, ...).filter(...)
    TrainDataset(train)  
}

val featurize = lift[TrainDataset, FeaturizedDataset] {
  case train => 
     // Compute featurization using ML Pipeline API
     val pipeline = new Pipeline()
       .setStages(Array(encodeSites, encodeS2Cells, assemble, lr))
     pipeline.fit(train.dataFrame).transform(train.dataFrame)  
}

val trainModel = lift[FeaturizedDataset, TrainedModel] {
  case featurized => 
    // Train model with featurized data 
    val lr = new LogisticRegression().set(...)
    val pipeline = new Pipeline().setStages(Array(encode, lr))
    val evaluator = new BinaryClassificationEvaluator()   
    val crossValidator = new CrossValidator()
      .setEstimator(pipeline)
      .setEvaluator(evaluator)    
    val paramGrid = new ParamGridBuilder()
      .addGrid(lr.elasticNetParam, Array(0.1, 0.5))
      .build()    
    val model = crossValidator.fit(featurized.dataFrame)
    TrainedModel(model)
}

val prepareTestDataset = lift[TrainedModel, TestDataset] {
  case model => ...
} 

val evaluateModel = lift[TestDataset, TrainedAndEvaluatedModel] {
  case test => ...
}

// Helper method that allows each step to 
// return `ModelingError \/ Result` and nicely chains it together
private def lift[A, B](f: A => ModelingError \/ B) = {
  channel.lift[Task, ModelingError \/ A, ModelingError \/ B] {
      case -\/(err) => Task.now(\/.left(err))
      case \/-(a) => task(f(a))
    }
}
{% endcoderay %}

Given previously defined modeling steps, optimization pipeline can be defined as `Process` transformation.

{% coderay lang:groovy %}
def optimize(
  campaigns: Process[Task, OptimizedCampaign]
): Process[Task, ModelingError \/ TrainedAndEvaluatedModel] = {

  campaigns
    .concurrently(2)(prepareResponse)
    .concurrently(2)(prepareTrainDataset)
    .concurrently(2)(featurize)
    .concurrently(10)(trainModel)
    .concurrently(2)(prepareTestDataset)
    .concurrently(2)(evaluateModel)
}
{% endcoderay %}

I'm using `concurrently` method, which runs each step with controlled concurrency in separate threads. Steps that are doing heavy shuffles 
are running not more than 2 in parallel, in contrast to model training that is relatively lightweight operation and can run with much higher
concurrency. This helper method is described in [earlier post](/blog/2015/09/09/audience-modeling-with-spark-ml-pipelines).


#### Push vs Pull Based Streams

`scalaz-steam` uses pull based model, it means that not first step (prepare response) is pushing data down the transformation chain when it's ready, but
the bottom step (evaluate model) asks the previous step for new data when it's done.

This allows to keep Spark cluster always busy, for example when relatively slow running modeling step is done, it 
asks `featurize` for new data, and it's already there, which means that modeling can start immediately.


### Spark Cluster Tuning

For better cluster utilization I suggest to use `FAIR` scheduler mode, that can be turned on with `--conf spark.scheduler.mode=FAIR` flag.

Another big problem for us was tuning garbage collection. I've spent a lot of time trying to tune `G1` collector, but `ConcMarkSweepGC` with `ParNewGC` showed the best results
in our case. It doesn't guarantee that it's also the best choice for your particular case.

{% coderay %}
--conf spark.executor.extraJavaOptions="-server -XX:+AggressiveOpts -XX:-UseBiasedLocking -XX:NewSize=4g -XX:MaxNewSize=4g -XX:+UseParNewGC -XX:MaxTenuringThreshold=2 -XX:SurvivorRatio=4 -XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=32768 -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+ParallelRefProcEnabled -XX:+CMSClassUnloadingEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:+PrintTenuringDistribution -XX:+PrintGCDateStamps"
--driver-java-options "-server -XX:+AggressiveOpts -XX:-UseBiasedLocking -XX:NewSize=4g -XX:MaxNewSize=4g -XX:+UseParNewGC -XX:MaxTenuringThreshold=2 -XX:SurvivorRatio=4 -XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=32768 -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+ParallelRefProcEnabled -XX:+CMSClassUnloadingEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:+PrintTenuringDistribution -XX:+PrintGCDateStamps -Xloggc:gc.log" 
{% endcoderay %}


### Streams Everywhere

`scalaz-stream` is a great abstraction and as it's described in [scalaz-stream Introduction](https://gist.github.com/djspiewak/d93a9c4983f63721c41c) *every* application *can*, and I believe **should be** modeled this way.

This approach is embraced not only in scala community, but also in clojure, take a look for example at 
Rich Hickey presentation about [clojure core.async channels](http://www.infoq.com/presentations/clojure-core-async), and how your application can
be modeled with queues.
