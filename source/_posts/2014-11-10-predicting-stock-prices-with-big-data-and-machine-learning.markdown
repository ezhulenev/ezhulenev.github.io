---
layout: post
title: "Predicting Stock Prices with Big Data and Machine Learning"
date: 2014-11-10 22:03:35 -0400
comments: true
categories: [spark, mllib, machine learning, finance, scala]
keywords: spark, mllib, machine learning, finance, scala
---

Spark and MLLib for building price movement prediction model based from order log data.

### Synopsis

This post is based on [Modeling high-frequency limit order book dynamics with support vector machines](https://raw.github.com/ezhulenev/scala-openbook/master/assets/Modeling-high-frequency-limit-order-book-dynamics-with-support-vector-machines.pdf) paper.
Roughly speaking I'm implementing ideas introduced in this paper in scala with [Spark](https://spark.apache.org/) and [Spark MLLib](https://spark.apache.org/mllib/).
Authors are using sampling, I'm going to use full order log from [NYSE](http://www.nyxdata.com/Data-Products/NYSE-OpenBook-History) (sample data is available from [NYSE FTP](ftp://ftp.nyxdata.com/Historical%20Data%20Samples/TAQ%20NYSE%20OpenBook/)), just because
I can easily do it with Spark.

If you want to get deep understanding of the problem and proposed solution, you need to read the paper. I'm going to describe it in less academic language in one or two paragraphs.

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
I'm not going to dive deep into details. Concepts are super easy for understanding.

> An electronic list of buy and sell orders for a specific security or financial instrument, organized by price level.