---
layout: post
title: "Stock price prediction with Big Data and Machine Learning"
date: 2014-11-10 22:03:35 -0400
comments: true
categories: [spark, mllib, machine learning, finance, scala]
keywords: spark, mllib, machine learning, finance, scala
---

Apache Spark and Spark MLLib for building price movement prediction model based from order log data.

> The code for this application app can be found on [Github](https://github.com/ezhulenev/svm-orderbook-dynamics)

### Synopsis

This post is based on [Modeling high-frequency limit order book dynamics with support vector machines](https://raw.github.com/ezhulenev/scala-openbook/master/assets/Modeling-high-frequency-limit-order-book-dynamics-with-support-vector-machines.pdf) paper.
Roughly speaking I'm implementing ideas introduced in this paper in scala with [Spark](https://spark.apache.org/) and [Spark MLLib](https://spark.apache.org/mllib/).
Authors are using sampling, I'm going to use full order log from [NYSE](http://www.nyxdata.com/Data-Products/NYSE-OpenBook-History) (sample data is available from [NYSE FTP](ftp://ftp.nyxdata.com/Historical%20Data%20Samples/TAQ%20NYSE%20OpenBook/)), just because
I can easily do it with Spark.

If you want to get deep understanding of the problem and proposed solution, you need to read the paper.
I'm going to give high level overview of the problem in less academic language, in one or two paragraphs.

> Predictive modelling is the process by which a model is created or chosen to try to best predict the probability of an outcome.

#### Model Architecture

Authors are proposing framework for extracting feature vectors from from raw order log data, that can be used as input to
machine learning method (SVM in particular) to predict price movement (Up, Down, Stationary). Given a set of training data
with assigned labels (price movement) SVM training algorithm builds a model that assigns new examples into one category or the other.

{% coderay %}
Time(sec)            Price($)   Volume      Event Type      Direction
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
34203.011926972      598.68     10          submission      ask
34203.011926973      594.47     15          submission      bid
34203.011926974      594.49     20          submission      bid
34203.011926981      597.68     30          submission      ask
34203.011926991      594.47     15          execution       ask
34203.011927072      597.68     10          cancellation    ask
34203.011927082      599.88     12          submission      ask
34203.011927097      598.38     11          submission      ask
{% endcoderay %}

In the table, each row of the message book represents a trading event that could be either a order submission,
order cancellation, or order execution. The arrival time measured from midnight,
is in seconds and nanoseconds; price is in US dollars, and the Volume is in number of shares.
Ask - I'm selling and asking for this price, Bid - I want to buy for this price.

From this log it's very easy to reconstruct state of order book after each entry. You can read more about [order book](http://www.investopedia.com/terms/o/order-book.asp)
and [limit order book](http://www.investopedia.com/university/intro-to-order-types/limit-orders.asp) in Investopedia,
I'm not going to dive into details. Concepts are super easy for understanding.

> An electronic list of buy and sell orders for a specific security or financial instrument, organized by price level.

#### Feature Extraction and Training Data Preparation

After order books are reconstructed from order log, we can derive attributes, that will form feature vectors used as input to `SVM`.

Attributes are divided into three categories: basic, time-insensitive, and time-sensitive, all of which can be directly computed from the data.
Attributes in the basic set are the prices and volumes at both ask and bid sides up to n = 10 different levels (that is, price levels in the order book at a given moment),
which can be directly fetched from the order book. Attributes in the time-insensitive set are easily computed from the basic set at a single point in time.
Of this, bid-ask spread and mid-price, price ranges, as well as average price and volume at different price levels are calculated in feature sets `v2`, `v3`, and `v5`, respectively;
while `v5` is designed to track the accumulated differences of price and volume between ask and bid sides. By further taking the recent history of current data into consideration,
we devise the features in the time-sensitive set. More about calculating other attributes can be found in [original paper](https://raw.github.com/ezhulenev/scala-openbook/master/assets/Modeling-high-frequency-limit-order-book-dynamics-with-support-vector-machines.pdf).

{%img center https://raw.github.com/ezhulenev/scala-openbook/master/assets/features.png %}

#### Learning Model Construction

To prepare training data for machine learning it's also required to label each point with price movement observed over some time horizon (1 second fo example).
It's straightforward task that only requires to know the state of order book after given time after current event.

In our case label set has size greater than 2 (Up, Down, Stationary), and instead of solving a multi-class categorization problem,
we reduce the multi-class learning problem into a set of binary classification tasks and build a binary classifier independently for each label
with a one-against-all method.

Luckily binary classification problem is already solved in [Spark MLLib](https://spark.apache.org/mllib/), and
I'm going to use [Linear support vector machines (SVMs)](http://spark.apache.org/docs/1.1.0/mllib-linear-methods.html) with L2 regularization.


### Order Log Data

I'm going to use [NYSE TAQ OpenBook](http://www.nyxdata.com/Data-Products/NYSE-OpenBook-History) orders data, and parse it with [Scala OpenBook](https://github.com/ezhulenev/scala-openbook)
library. It's easiest data set to get, free sample data for 2 trading days is available for download at [NYSE FTP](ftp://ftp.nyxdata.com/Historical%20Data%20Samples/TAQ%20NYSE%20OpenBook/).

> TAQ (Trades and Quotes) historical data products provide a varying range of market depth on a T+1 basis for covered markets.
> TAQ data products are used to develop and backtest trading strategies, analyze market trends as seen in a real-time ticker plant environment, and research markets for regulatory or audit activity.

### Prepare Training Data with Spark

#### Extract Feature Vectors

#### Label Training Data

### Build Classification Model

#### Feature Selection

#### Model Performance with Tests of Significance

### Results

I was able to relatively easy reproduce fairly complicated research and machine learning problem at large scale.

Leveraging latest Big Data technologies, and Apache Spark specifically, allows switch from building models
using sampling and only small fraction of data, to iterative research workflow with all the data you can get access to.

> The code for this application app can be found on [Github](https://github.com/ezhulenev/svm-orderbook-dynamics)
