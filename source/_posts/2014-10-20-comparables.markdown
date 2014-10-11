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


### Approaches that we tried

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


### Data Model

We use Cassandra to store all mentions, aggregates and recommendations. We use [Phantom DSL](https://github.com/websudos/phantom)
for scala to define Cassandra schema and build services.

{% codeblock lang:scala %}
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
{% endcodeblock %}


### Spark Job For Aggregation and Recommendation

We use [Datastax Spark Cassandra Connector](https://github.com/datastax/spark-cassandra-connector)
to read raw mentions data from Cassandra into Spark RDD. After data is available as `RDD[Mention]` we run Spark job
to build aggregates and later another job to derive recommendations from aggregates.