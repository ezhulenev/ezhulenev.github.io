---
layout: post
title: "Building Twitter live stream analytics with Spark and Cassandra"
date: 2014-10-20 21:01:15 -0400
comments: true
categories: [spark, spark-streaming, cassandra, twitter, finance, scala]
keywords: spark, spark-streaming, cassandra, twitter, finance, scala
---

### Background

At [Pellucid Analytics](http://pellucid.com) we build platform that automates and simplifies
creation data-driven chartbooks, so it takes minutes rather than hours to go from raw data to powerful
visualizations and compelling stories.

One of industries in our focus is Investment Banking. We help IB advisory professionals build pitch-books, provide them
analytical and quantitative support to sell their ideas. [Comparable Companies Analysis](http://www.investopedia.com/terms/c/comparable-company-analysis-cca.asp) is
central to this business.

> Comparable company analysis starts with establishing a peer group consisting of similar companies of similar size in the same industry and region.

We faced with a problem of finding scalable solution to establish a peer group for any company.


### Approaches That We Tried

#### Company Industry

Data vendors provide [industry classification](http://en.wikipedia.org/wiki/Industry_classification) for each company,
and it helps a lot in industries like retail (Wal-Mart is good comparable to Costco), energy (Chevron and Exxon Mobil)
but it stumbles with many other companies. People tend to compare Amazon with Google as a two major players in it business,
but data vendors tend to put Amazon in retail industry with Wal-Mart/Costco as comparables.

#### Company Financials and Valuation Multiples

We tried cluster analysis and k-nearest neighbors to group companies based on their financials (Sales, Revenue)
and valuation multiples (EV/EBIDTA, P/E). However assumption that similar companies will have similar
valuations multiples is wrong. People compare Twitter with Facebook as two biggest companies in social media, but
based on their financials they don't have too much in common. Facebook 2013 revenue is almost $8 billions
and Twitter has only $600 millions.

### Novel Approach

We came up with idea, that if companies often mentioned in news, articles and tweets together, probably it's a sign
that people this about them as comparable companies. In this post I'll show how we built proof of concept for this idea
with Spark, Spark Streaming and Cassandra. We use only Twitter live stream data for now, accessing high quality news data is
a bit more complicated problem.

<!-- more -->

Let's take for example this tweet from CNN:

<blockquote class="twitter-tweet" lang="en"><p>Trying to spot the next <a href="https://twitter.com/search?q=%24FB&amp;src=ctag">$FB</a> or <a href="https://twitter.com/search?q=%24TWTR&amp;src=ctag">$TWTR</a>? These 10 startups are worth keeping an eye on <a href="http://t.co/FEKNtm7QqB">http://t.co/FEKNtm7QqB</a></p>&mdash; CNN Public Relations (@CNNPR) <a href="https://twitter.com/CNNPR/status/518083527863435264">October 3, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

From this tweet we can derive 2 mentions for 2 companies. For Facebook it will be Twitter and vice-versa. If we collect
tweets for all companies over some period of time, and take a ratio of joint appearance in same tweet as a
measure of "similarity", we can build comparable company recommendations based on this measure.

### Data Model

We use [Cassandra](http://cassandra.apache.org/) to store all mentions, aggregates and final recommendations.
We use [Phantom DSL](https://github.com/websudos/phantom) for scala to define schema
and for most of Cassandra operations (spark integration is not yet supported in Phantom).

{% coderay lang:groovy %}
/**
 * Mention of focus company
 *
 * @param ticker   ticker of focus company
 * @param source   source of this mention (Twitter, RSS, etc...)
 * @param sourceId source specific id
 * @param time     time
 * @param mentions set of other tickers including focus ticker itself
 */
case class Mention(ticker: Ticker, source: String, sourceId: String, time: DateTime, mentions: Set[Ticker])

sealed class MentionRecord extends CassandraTable[MentionRecord, Mention] with Serializable {

  override val tableName: String = "mention"

  object ticker    extends StringColumn    (this)  with PartitionKey[String]
  object source    extends StringColumn    (this)  with PrimaryKey[String]
  object time      extends DateTimeColumn  (this)  with PrimaryKey[DateTime]
  object source_id extends StringColumn    (this)  with PrimaryKey[String]
  object mentions  extends SetColumn[MentionRecord, Mention, String] (this)

  def fromRow(r: Row): Mention = {
    Mention(Ticker(ticker(r)), source(r), source_id(r), time(r), mentions(r) map Ticker)
  }
}
{% endcoderay %}


{% coderay lang:groovy %}
/**
 * Count mentions for each ticker pair
 *
 * @param ticker        ticker of focus company
 * @param mentionedWith mentioned with this ticker
 * @param count         number of mentions
 */
case class MentionsAggregate(ticker: Ticker, mentionedWith: Ticker, count: Long)

sealed class MentionsAggregateRecord extends CassandraTable[MentionsAggregateRecord, MentionsAggregate] {

  override val tableName: String = "mentions_aggregate"

  object ticker         extends StringColumn (this) with PartitionKey[String]
  object mentioned_with extends StringColumn (this) with PrimaryKey[String]
  object counter        extends LongColumn   (this)

  def fromRow(r: Row): MentionsAggregate = {
    MentionsAggregate(Ticker(ticker(r)), Ticker(mentioned_with(r)), counter(r))
  }
}
{% endcoderay %}

{% coderay lang:groovy %}
/**
 * Recommendation built based on company mentions with other companies
 *
 * @param ticker         focus company ticker
 * @position             recommendation position
 * @param recommendation recommended company ticker
 * @param p              number of times recommended company mentioned together
 *                       with focus company divided by total focus company mentions
 */
case class Recommendation(ticker: Ticker, position: Long, recommendation: Ticker, p: Double)

sealed class RecommendationRecord extends CassandraTable[RecommendationRecord, Recommendation] {

  override val tableName: String = "recommendation"

  object ticker         extends StringColumn (this) with PartitionKey[String]
  object position       extends LongColumn   (this) with PrimaryKey[Long]
  object recommendation extends StringColumn (this)
  object p              extends DoubleColumn (this)

  def fromRow(r: Row): Recommendation = {
    Recommendation(Ticker(ticker(r)), position(r), Ticker(recommendation(r)), p(r))
  }
}
{% endcoderay %}


### Ingest Real-Time Twitter Stream

We use [Spark Streaming](https://spark.apache.org/streaming/) Twitter integration to subscribe for
real-time twitter updates, then we extract company mentions and put them to Cassandra. Unfortunately Phantom
doesn't support Spark yet, so we used [Datastax Spark Cassandra Connector](https://github.com/datastax/spark-cassandra-connector)
with custom type mappers to map from Phantom-record types into Cassandra tables.

{% coderay lang:groovy %}
class MentionStreamFunctions(@transient stream: DStream[Mention]) extends Serializable {

  import TickerTypeConverter._

  TypeConverter.registerConverter(StringToTickerTypeConverter)
  TypeConverter.registerConverter(TickerToStringTypeConverter)

  implicit object MentionMapper extends DefaultColumnMapper[Mention](Map(
    "ticker"        -> "ticker",
    "source"        -> "source",
    "sourceId"      -> "source_id",
    "time"          -> "time",
    "mentions"      -> "mentions"
  ))

  def saveMentionsToCassandra(keyspace: String) = {
    stream.saveToCassandra(keyspace, MentionRecord.tableName)
  }
}
{% endcoderay %}


{% coderay lang:groovy %}
  private val filters = Companies.load().map(c => s"$$${c.ticker.value}")

  val sc = new SparkContext(sparkConf)
  val ssc = new StreamingContext(sc, Seconds(2))

  val stream = TwitterUtils.createStream(ssc, None, filters = filters)

  // Save Twitter Stream to cassandra
  stream.foreachRDD(updates => log.info(s"Received Twitter stream updates. Count: ${updates.count()}"))
  stream.extractMentions.saveMentionsToCassandra(keySpace)

  // Start Streaming Application
  ssc.start()
{% endcoderay %}


### Spark For Aggregation and Recommendation

To come up with comparable company recommendation we use 2-step process.

##### 1. Count mentions for each pair of tickers

After `Mentions` table loaded in Spark as `RDD[Mention]` we extract pairs of tickers,
and it enables bunch of aggregate and reduce functions from Spark `PairRDDFunctions`.
With `aggregateByKey` and given combine functions we efficiently build counter map `Map[Ticker, Long]` for each
ticker distributed in cluster. From single `Map[Ticker, Long]` we emit multiple aggregates for each ticket pair.

{% coderay lang:groovy %}
class AggregateMentions(@transient sc: SparkContext, keyspace: String)
  extends CassandraMappers with Serializable {

  private type Counter = Map[Ticker, Long]

  private implicit lazy val summ = Semigroup.instance[Long](_ + _)

  private lazy val seqOp: (Counter, Ticker) => Counter = {
    case (counter, ticker) if counter.isDefinedAt(ticker) => counter.updated(ticker, counter(ticker) + 1)
    case (counter, ticker) => counter + (ticker -> 1)
  }

  private lazy val combOp: (Counter, Counter) => Counter = {
    case (l, r) => implicitly[Monoid[Counter]].append(l, r)
  }

  def aggregate(): Unit = {
    // Emit pairs of (Focus Company Ticker, Mentioned With)
    val pairs = sc.cassandraTable[Mention](keyspace, MentionRecord.tableName).
      flatMap(mention => mention.mentions.map((mention.ticker, _)))

    // Calculate mentions for each ticker
    val aggregated = pairs.aggregateByKey(Map.empty[Ticker, Long])(seqOp, combOp)

    // Build MentionsAggregate from counters
    val mentionsAggregate = aggregated flatMap {
      case (ticker, counter) => counter map {
        case (mentionedWith, count) => MentionsAggregate(ticker, mentionedWith, count)
      }
    }

    mentionsAggregate.saveToCassandra(keyspace, MentionsAggregateRecord.tableName)
  }
}
{% endcoderay %}

##### 2. Sort aggregates and build recommendations

After aggregates computed, we sort them globally and then group them by key (Ticker). After
all aggregates grouped we produce `Recommendation` in single traverse distributed for each key.

{% coderay lang:groovy %}
class Recommend(@transient sc: SparkContext, keyspace: String)
  extends CassandraMappers with Serializable {

  private def toRecommendation: (MentionsAggregate, Int) => Recommendation = {
    var totalMentions: Option[Long] = None

    {
      case (aggregate, idx) if totalMentions.isEmpty =>
        totalMentions = Some(aggregate.count)
        Recommendation(aggregate.ticker, idx, aggregate.mentionedWith, 1)

      case (aggregate, idx) =>
        Recommendation(aggregate.ticker, idx,
                       aggregate.mentionedWith,
                       aggregate.count.toDouble / totalMentions.get)
    }
  }

  def recommend(): Unit = {
    val aggregates = sc.
               cassandraTable[MentionsAggregate](keyspace, MentionsAggregateRecord.tableName).
               sortBy(_.count, ascending = false)

    val recommendations = aggregates.
      groupBy(_.ticker).
      mapValues(_.zipWithIndex).
      flatMapValues(_ map toRecommendation.tupled).values

    recommendations.saveToCassandra(keyspace, RecommendationRecord.tableName)
  }
}
{% endcoderay %}


### Results

You can check comparable company recommendations build from Twitter stream on Pellucid [web site](http://comparables.io.pellucid.com).

Cassandra and Spark works perfectly together and allows to build scalable data-driven applications,
that super easy to scale out and handle gigabytes and terabytes of data. In this particular case, it's
probably an overkill. Twitter doesn't have enough finance-related activity to produce serious load. However it's easy to
extend this application and add other streams: Bloomberg News Feed, Thompson Reuters, etc.

> The code for this application app can be found on [Github](https://github.com/pellucidanalytics/hackday)
