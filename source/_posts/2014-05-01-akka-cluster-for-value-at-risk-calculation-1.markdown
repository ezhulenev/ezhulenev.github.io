---
layout: post
title: "Akka Cluster for Value-at-Risk calculation (Part 1/2)"
date: 2014-05-01 22:03:45 -0400
comments: true
categories: [scala, akka, cluster, value-at-risk, grid]
keywords: scala, akka, cluster, value-at-risk, grid, grid-computing
---

### Synopsis

> The code & sample app can be found on [Github](https://github.com/ezhulenev/akka-var-calculation)

Risk Management in finance is one of the most common [case studies](http://www.gridgain.com/usecases/risk-management/)
for Grid Computing, and Value-at-Risk is most widely used risk measure.
In this article I'm going to show how to `scale-out` Value-at-Risk calculation to multiple nodes with latest [Akka](http://akka.io) middleware.
In Part 1 I'm describing the problem and `single-node` solution, and in Part 2 I'm scaling it to multiple nodes.


## [Part 1] Introduction to Value at Risk calculation

Go to [Part 2](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-2) where VaR calculation scaled-out to multiple nodes.

<!-- more -->

### What is Value-at-Risk

> In financial mathematics and financial risk management, value at risk (VaR) is a widely used risk measure of the risk of loss on a specific portfolio of financial assets. For a given portfolio, probability and time horizon, VaR is defined as a threshold value such that the probability that the mark-to-market loss on the portfolio over the given time horizon exceeds this value (assuming normal markets and no trading in the portfolio) is the given probability level.

You can continue with reading amazing set of articles ["Introduction to Grid Computing for Value At Risk calculation"](http://blog.octo.com/en/introduction-to-grid-computing-for-value-at-risk-calculatio/) where VaR described in more details
and explained why it's a perfect fit for grid computing and spreading calculation across multiple nodes. Also there you can find examples for [GridGain](http://www.gridgain.com/) and [Hadoop](http://hadoop.apache.org/). In this article I'm going to use Akka.

### Next step

In next part I'm going to scale-out VaR calculation to multiple nodes, and I will be using latest Akka Cluster for it.

## [>>> Go to Part 2](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-2)