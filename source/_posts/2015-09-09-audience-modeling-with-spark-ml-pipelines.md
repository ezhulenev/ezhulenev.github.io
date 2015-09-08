---
layout: post
title: "Audience modeling with Spark ML Pipelines"
date: 2015-09-09 08:04:25 -0500
comments: true
categories: [spark, scala, dataframe, machine learning, audience modeling]
keywords: spark, scala, dataframe, machine learning, audience modeling
---

At [Collective](http://collective.com) we are heavily relying on machine learning and predictive modeling to 
run digital advertising business. All decisions about what ad to show at this particular time to this particular user
are made by machine learning models (some of them are real time, and some of them are offline).

We have a lot of projects that uses machine learning, common name for all of them can be **Audience Modeling**, as they
all are trying to predict audience conversion (*CTR, Viewability Rate, etc...*) based on browsing history, behavioral segments and other type of 
predictors.

For most of new development we use [Spark](https://spark.apache.org) and [Spark MLLib](https://spark.apache.org/mllib/). It is a awesome project,
however we found that some nice tools/libraries that are widely used for example in R are missing in Spark. In order to add missing
features that we would really like to have in Spark, we created [Spark Ext](https://github.com/collectivemedia/spark-ext) - Spark Extensions
Library. 

> Spark Ext on Github: [https://github.com/collectivemedia/spark-ext](https://github.com/collectivemedia/spark-ext)

I'm going to show simple example of combining [Spark Ext](https://github.com/collectivemedia/spark-ext) with Spark ML pipelines for predicting user conversions based geo and browsing history data.

> Spark ML pipeline example: [SparkMlExtExample.scala](https://github.com/collectivemedia/spark-ext/blob/master/sparkext-example/src/main/scala/com/collective/sparkext/example/SparkMlExtExample.scala)

<!-- more -->

## Predictors Data

I'm using dataset with 2 classes, that will be used for solving classification problem (user converted or not). It's created with 
[dummy data generator](https://github.com/collectivemedia/spark-ext/blob/master/sparkext-example/src/main/scala/com/collective/sparkext/example/DataGenerator.scala), 
so that these 2 classes can be easily separated. It's pretty similar to real data that usually available in digital advertising.
 
### Browsing History Log

History of web sites that were visited by user.

{% coderay %}
Cookie          | Site          | Impressions  
--------------- |-------------- | -------------
wKgQaV0lHZanDrp | live.com      | 24
wKgQaV0lHZanDrp | pinterest.com | 21
rfTZLbQDwbu5mXV | wikipedia.org | 14
rfTZLbQDwbu5mXV | live.com      | 1
rfTZLbQDwbu5mXV | amazon.com    | 1
r1CSY234HTYdvE3 | youtube.com   | 10
{% endcoderay %}

### Geo Location Log

Latitude/Longitude impression history.

{% coderay %}
Cookie          | Lat     | Lng       | Impressions
--------------- |---------| --------- | ------------
wKgQaV0lHZanDrp | 34.8454 | 77.009742 | 13
wKgQaV0lHZanDrp | 31.8657 | 114.66142 | 1
rfTZLbQDwbu5mXV | 41.1428 | 74.039600 | 20
rfTZLbQDwbu5mXV | 36.6151 | 119.22396 | 4
r1CSY234HTYdvE3 | 42.6732 | 73.454185 | 4
r1CSY234HTYdvE3 | 35.6317 | 120.55839 | 5
20ep6ddsVckCmFy | 42.3448 | 70.730607 | 21
20ep6ddsVckCmFy | 29.8979 | 117.51683 | 1
{% endcoderay %}


## Transforming Predictors Data

As you can see predictors data (sites and geo) is in *long* format, each `cookie` has multiple rows associated with it,
and it's in general is not a good fit for machine learning.
We'd like `cookie` to be a primary key, and all other data should form `feature vector`.

### Gather Transformer

Inspired by R `tidyr` and `reshape2` packages. Convert *long* `DataFrame` with values
for each key into *wide* `DataFrame`, applying aggregation function if single
key has multiple values.

{% coderay lang:groovy %}
val gather = new Gather()
      .setPrimaryKeyCols("cookie")
      .setKeyCol("site")
      .setValueCol("impressions")
      .setValueAgg("sum")         // sum impression by key
      .setOutputCol("sites")
val gatheredSites = gather.transform(siteLog)      
{% endcoderay %}

{% coderay %}
Cookie           | Sites
-----------------|----------------------------------------------
wKgQaV0lHZanDrp  | [
                 |  { site: live.com, impressions: 24.0 }, 
                 |  { site: pinterest.com, impressions: 21.0 }
                 | ]
rfTZLbQDwbu5mXV  | [
                 |  { site: wikipedia.org, impressions: 14.0 }, 
                 |  { site: live.com, impressions: 1.0 },
                 |  { site: amazon.com, impressions: 1.0 }
                 | ]
{% endcoderay %}

### Google S2 Geometry Cell Id Transformer

The S2 Geometry Library is a spherical geometry library, very useful for manipulating regions on the sphere (commonly on Earth) 
and indexing geographic data. Basically it assigns unique cell id for each region on the earth. 

> Good article about S2 library: [Google’s S2, geometry on the sphere, cells and Hilbert curve](http://blog.christianperone.com/2015/08/googles-s2-geometry-on-the-sphere-cells-and-hilbert-curve/)

For example you can combine S2 transformer with Gather to get from `lat`/`lon` to `K-V` pairs, where key will be `S2` cell id.
Depending on a level you can assign all people in Greater New York area (level = 4) into one cell, or you can index them block by block (level = 12).

{% coderay lang:groovy %}
// Transform lat/lon into S2 Cell Id
val s2Transformer = new S2CellTransformer()
  .setLevel(5)
  .setCellCol("s2_cell")

// Gather S2 CellId log
val gatherS2Cells = new Gather()
  .setPrimaryKeyCols("cookie")
  .setKeyCol("s2_cell")
  .setValueCol("impressions")
  .setOutputCol("s2_cells")
  
val gatheredCells = gatherS2Cells.transform(s2Transformer.transform(geoDf))
{% endcoderay %}

{% coderay %}
Cookie           | S2 Cells
-----------------|----------------------------------------------
wKgQaV0lHZanDrp  | [
                 |  { s2_cell: d5dgds, impressions: 5.0 }, 
                 |  { s2_cell: b8dsgd, impressions: 1.0 }
                 | ]
rfTZLbQDwbu5mXV  | [
                 |  { s2_cell: d5dgds, impressions: 12.0 }, 
                 |  { s2_cell: b8dsgd, impressions: 3.0 },
                 |  { s2_cell: g7aeg3, impressions: 5.0 }
                 | ]
{% endcoderay %}


## Assembling Feature Vector

`K-V` pairs from result of `Gather` are cool, and groups all the information about cookie into single row, however they can't be used
as input for machine learning. To be able to train a model, predictors data needs to be represented as a vector of doubles. If all features are continuous and
numeric it's easy, but if some of them are categorical or in `gathered` shape, it's not trivial.

### Gather Encoder

Encodes categorical key-value pairs using dummy variables. 

{% coderay lang:groovy %}
// Encode S2 Cell data
val encodeS2Cells = new GatherEncoder()
  .setInputCol("s2_cells")
  .setOutputCol("s2_cells_f")
  .setKeyCol("s2_cell")
  .setValueCol("impressions")
  .setCover(0.95) // dimensionality reduction
{% endcoderay %}
 

{% coderay %}
Cookie           | S2 Cells
-----------------|----------------------------------------------
wKgQaV0lHZanDrp  | [
                 |  { s2_cell: d5dgds, impressions: 5.0 }, 
                 |  { s2_cell: b8dsgd, impressions: 1.0 }
                 | ]
rfTZLbQDwbu5mXV  | [
                 |  { s2_cell: d5dgds, impressions: 12.0 }, 
                 |  { s2_cell: g7aeg3, impressions: 5.0 }
                 | ]
{% endcoderay %}

Transformed into

{% coderay %}
Cookie           | S2 Cells Features
-----------------|------------------------
wKgQaV0lHZanDrp  | [ 5.0  ,  1.0 , 0   ]
rfTZLbQDwbu5mXV  | [ 12.0 ,  0   , 5.0 ]
{% endcoderay %}

Note that it's 3 unique cell id values, that gives 3 columns in final feature vector.

Optionally apply dimensionality reduction using `top` transformation:

 - Top coverage, is selecting categorical values by computing the count of distinct users for each value,
   sorting the values in descending order by the count of users, and choosing the top values from the resulting
   list such that the sum of the distinct user counts over these values covers c percent of all users,
   for example, selecting top sites covering 99% of users.
   
## Spark ML Pipelines
   
Spark ML Pipeline - is new high level API for Spark MLLib. 
 
> A practical ML pipeline often involves a sequence of data pre-processing, feature extraction, model fitting, and validation stages. For example, classifying text documents might involve text segmentation and cleaning, extracting features, and training a classification model with cross-validation. [Read More.](https://databricks.com/blog/2015/01/07/ml-pipelines-a-new-high-level-api-for-mllib.html) 

In Spark ML it's possible to split ML pipeline in multiple independent stages, group them together in single pipeline and run it
with Cross Validation and Parameter Grid to find best set of parameters.

### Put It All together with Spark ML Pipelines

Gather encoder is a natural fit into Spark ML Pipeline API.

{% coderay lang:groovy %}
// Encode site data
val encodeSites = new GatherEncoder()
  .setInputCol("sites")
  .setOutputCol("sites_f")
  .setKeyCol("site")
  .setValueCol("impressions")

// Encode S2 Cell data
val encodeS2Cells = new GatherEncoder()
  .setInputCol("s2_cells")
  .setOutputCol("s2_cells_f")
  .setKeyCol("s2_cell")
  .setValueCol("impressions")
  .setCover(0.95)

// Assemble feature vectors together
val assemble = new VectorAssembler()
  .setInputCols(Array("sites_f", "s2_cells_f"))
  .setOutputCol("features")

// Build logistic regression
val lr = new LogisticRegression()
  .setFeaturesCol("features")
  .setLabelCol("response")
  .setProbabilityCol("probability")

// Define pipeline with 4 stages
val pipeline = new Pipeline()
  .setStages(Array(encodeSites, encodeS2Cells, assemble, lr))

val evaluator = new BinaryClassificationEvaluator()
  .setLabelCol(Response.response)

val crossValidator = new CrossValidator()
  .setEstimator(pipeline)
  .setEvaluator(evaluator)

val paramGrid = new ParamGridBuilder()
  .addGrid(lr.elasticNetParam, Array(0.1, 0.5))
  .build()

crossValidator.setEstimatorParamMaps(paramGrid)
crossValidator.setNumFolds(2)

println(s"Train model on train set")
val cvModel = crossValidator.fit(trainSet)
{% endcoderay %}


## Conclusion

New Spark ML API makes machine learning much more easier. [Spark Ext](https://github.com/collectivemedia/spark-ext) is good example of how is it possible to 
create custom transformers/estimators that later can be used as a part of bigger pipeline, and can be easily shared/reused by multiple projects.

> Full code for example application is available on [Github](https://github.com/collectivemedia/spark-ext/blob/master/sparkext-example/src/main/scala/com/collective/sparkext/example/SparkMlExtExample.scala).
