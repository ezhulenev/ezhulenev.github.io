---
layout: post
title: "Akka Cluster for Value-at-Risk calculation (Part 2/2)"
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


## [Part 2/2] Scale-out VaR calculation to multiple nodes

Go to [Part 1](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-1) where VaR calculation problem is defined.

### Akka Cluster


## [<<< Go to Part 1](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-1)
