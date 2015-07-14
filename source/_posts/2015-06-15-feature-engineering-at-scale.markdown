---
layout: post
title: "Feature Engineering at Scale with Spark"
date: 2015-06-15 22:02:45 -0500
comments: true
categories: [spark, scala, machine learning]
keywords: spark, scala, machine learning
---

> Check Model Matrix [Website](http://collectivemedia.github.io/modelmatrix/) and [Github project](https://github.com/collectivemedia/modelmatrix).

At [Collective](http://collective.com) we are in programmatic advertisement business, it means that all our
advertisement decisions (what ad to show, to whom and at what time) are driven by models. We do a lot of 
machine learning, build thousands predictive models and use them to make millions decision per second.

#### How do we get the most out of our data for predictive modeling?

Success of all Machine Learning algorithms depends on data that you put into it, the better the features you choose, the
better the results you will achieve.

> Feature Engineering is the process of using domain knowledge of the data to create features that make machine learning algorithms work better.

In Ad-Tech it's finite pieces of information about users that we can put into our models, and it's 
almost the same across all companies in industry, we don't have access to any anonymous data
like real name and age, interests on Facebook etc. It really matter how creative you are to get maximum from the data you have,
and how fast you can iterate and test new idea.

In 2014 Collective data science team published [Machine Learning at Scale](http://arxiv.org/abs/1402.6076) paper that
describes our approach and trade-offs for audience optimization. In 2015 we solve the same problems, but
using new technologies (Spark and Spark MLLib) at even bigger scale. I want to show the tool that I built specifically 
to handle feature engineering/selection problem, and which is open sources now.

## Model Matrix

<!-- more -->

### Feature Transformation

Imagine impression log that is used to train predictive model

{% coderay %}
visitor_id  | ad_campaign     | ad_id | ad_ctr     | pub_site            | state | city         | price | timestamp     
----------- | --------------- | ----- | ---------- | ------------------- | ----- | ------------ | ----- | ------------- 
bob         | Nike_Sport      | 1     | 0.01       | http://bbc.com      | NY    | New York     | 0.17  | 1431032702135  
bill        | Burgers_Co      | 2     | 0.005      | http://cnn.com      | CA    | Los Angeles  | 0.42  | 1431032705167 
mary        | Macys           | 3     | 0.015      | http://fashion.com  | CA    | Los Angeles  | 0.19  | 1431032708384 
{% endcoderay %}

Producing a feature vector for every visitor (cookie) row and every piece of information about a 
visitor as an p-size vector, where p is the number of predictor variables multiplied by cardinality 
of each variable (number of states in US, number of unique websites, etc ...). It is impractical 
both from the data processing standpoint and because the resulting vector would only have 
about 1 in 100,000 non-zero elements.

{% coderay %}
 visitor_id  | Nike_Sport | Burgers_Co | Macys | NY  | CA  | ... 
 ----------- | ---------- | ---------- | ----- | --- | --- | --- 
 bob         | 1.0        |            |       | 1.0 |     | ... 
 bill        |            | 1.0        |       |     | 1.0 | ... 
 mary        |            |            | 1.0   |     | 1.0 | ... 
{% endcoderay %}

Model Matrix uses feature transformations (top, index, binning) to reduce dimensionality to arrive 
at between one and two thousand predictor variables, with data sparsity of about 1 in 10. It removes 
irrelevant and low frequency predictor values from the model, and transforms continuous 
variable into bins of the same size.
  
{% coderay %}   
 visitor_id  | Nike | OtherAd | NY  | OtherState | price ∈ [0.01, 0.20) | price ∈ [0.20, 0.90) | ... 
 ----------- | ---- | ------- | --- | ---------- | -------------------- | -------------------- | --- 
 bob         | 1.0  |         | 1.0 |            | 1.0                  |                      | ... 
 bill        |      | 1.0     |     | 1.0        |                      | 1.0                  | ... 
 mary        |      | 1.0     |     | 1.0        |                      | 1.0                  | ... 
{% endcoderay %}

Transformation definitions in scala:

{% coderay lang:groovy %}
sealed trait Transform

/**
 * Absence of transformation
 */
case object Identity extends Transform

/**
 * For distinct values of the column, find top values
 * by a quantity that cumulatively cover a given percentage
 * of this quantity. For example, find the top DMAs that
 * represent 99% of cookies, or find top sites that
 * are responsible for 90% of impressions.
 *
 * @param cover      cumulative cover percentage
 * @param allOther   include feature for all other values
 */
case class Top(cover: Double, allOther: Boolean) extends Transform

/**
 * For distinct values of the column, find the values
 * with at least the minimum support in the data set.
 * Support for a value is defined as the percentage of a
 * total quantity that have that value. For example,
 * find segments that appear for at least 1% of the cookies.
 *
 * @param support    support percentage
 * @param allOther   include feature for all other values
 */
case class Index(support: Double, allOther: Boolean) extends Transform

/**
 * Break the values in the column into bins with roughly the same number of points.
 *
 * @param nbins target number of bins
 * @param minPoints minimum number of points in single bin
 * @param minPercents minimum percent of points in a bin (0-100).
 *                    The larger of absolute number and percent points is used.
 */
case class Bins(nbins: Int, minPoints: Int = 0, minPercents: Double = 0.0) extends Transform
{% endcoderay %}


### Transformed Columns

#### Categorical Transformation

A column calculated by applying top or index transform function, each columns id corresponds 
to one unique value from input data set. SourceValue is encoded as ByteVector unique value from 
input column and used later for featurization. 

{% coderay lang:groovy %}
class CategoricalTransformer(
  features: DataFrame @@ Transformer.Features
) extends Transformer(features) {

  def transform(feature: TypedModelFeature): Seq[CategoricalColumn]
  
}
{% endcoderay %}

{% coderay lang:groovy %}
sealed trait CategoricalColumn {
  def columnId: Int
  def count: Long
  def cumulativeCount: Long
}

object CategoricalColumn {

  case class CategoricalValue(
    columnId: Int,
    sourceName: String,
    sourceValue: ByteVector,
    count: Long,
    cumulativeCount: Long
  ) extends CategoricalColumn 

  case class AllOther(
    columnId: Int,
    count: Long,
    cumulativeCount: Long
  ) extends CategoricalColumn 
  
}
{% endcoderay %}

#### Bin Column
  
A column calculated by applying binning transform function.

{% coderay lang:groovy %}
class BinsTransformer(
  input: DataFrame @@ Transformer.Features
) extends Transformer(input) with Binner {

  def transform(feature: TypedModelFeature): Seq[BinColumn] = {
  
}
{% endcoderay %}

{% coderay lang:groovy %}
case class BinValue(
    columnId: Int,
    low: Double,
    high: Double,
    count: Long,
    sampleSize: Long
  ) 
{% endcoderay %}


### Building Model Matrix Instance

Model Matrix instance contains information about shape of the training data, what transformations (categorical and binning)
are required to apply to input data in order to obtain feature vector that will got into machine learning
algorithm.

Building model matrix instance described well in [command line interface documentation](http://collectivemedia.github.io/modelmatrix/doc/cli.html).

### Featurizing your data

When you have model matrix instance, you can apply it to multiple input data sets. For example in Collective
we build model matrix instance once a week or even month, and use it for building models from daily/hourly data.
It gives us nice property: all models have the same columns, and it's easy to compare them.
 
{% coderay lang:groovy %}

// Similar to Spark LabeledPoint
case class IdentifiedPoint(id: Any, features: Vector)

class Featurization(features: Seq[ModelInstanceFeature]) extends Serializable {

  // Check that all input features belong to the same model instance
  private val instances = features.map(_.modelInstanceId).toSet
  require(instances.size == 1, 
    s"Features belong to different model instances: $instances")

  // Maximum columns id in instance features
  private val totalNumberOfColumns = features.flatMap {
    case ModelInstanceIdentityFeature(_, _, _, _, columnId) => Seq(columnId)
    case ModelInstanceTopFeature(_, _, _, _, cols) => cols.map(_.columnId)
    case ModelInstanceIndexFeature(_, _, _, _, cols) => cols.map(_.columnId)
    case ModelInstanceBinsFeature(_, _, _, _, cols) => cols.map(_.columnId)
  }.max


  /**
   * Featurize input dataset
   *
   * @return id data type and featurized rows
   */
  def featurize(
    input: DataFrame @@ FeaturesWithId, 
    idColumn: String
  ): (DataType, RDD[IdentifiedPoint]) = {
  
    log.info(s"Extract features from input DataFrame with id column: $idColumn. " + 
             s"Total number of columns: $totalNumberOfColumns")
    
    ...
    
  }
}
{% endcoderay %}
 
 
### Results
 
Model Matrix is open sourced, and available on [Github](https://github.com/collectivemedia/modelmatrix), lot's of 
documentation on [Website](http://collectivemedia.github.io/modelmatrix/).

We use it at [Collective](http://collective.com) to define our models and it works for us really well.

You can continue your reading with [Machine Learning at Scale](http://arxiv.org/abs/1402.6076) paper, 
to get more data science focused details about our modeling approach.