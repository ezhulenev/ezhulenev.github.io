---
layout: post
title: "Akka Cluster for Value at Risk calculation (Part 1/2)"
date: 2014-05-01 22:03:45 -0400
comments: true
categories: [scala, akka, cluster, value-at-risk, grid, scalaz-stream]
keywords: scala, akka, cluster, value-at-risk, grid, grid-computing, scalaz-stream
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

### Data Model

Let's define simple model for supported instruments:

{% codeblock lang:scala %}
sealed trait Instrument

case class Equity(ticker: String) extends Instrument

sealed trait EquityOption extends Instrument {
  def underlying: Equity
}

case class CallOption(
           underlying: Equity,
           strike: Double,
           maturity: LocalDate) extends EquityOption

case class PutOption(
           underlying: Equity,
           strike: Double,
           maturity: LocalDate) extends EquityOption
{% endcodeblock %}

and portfolio for which market risk will be calculated:

{% codeblock lang:scala %}
 case class Position(instrument: Instrument, n: Int)

 case class Portfolio(positions: NonEmptyList[Position])
{% endcodeblock %}

### Portfolio pricing

Let's define elementary portfolio pricing service, that can calculate market price for instrument based on market factors:

{% codeblock lang:scala%}
sealed trait MarketFactor

object MarketFactor {
  case class Price(equity: Equity) extends MarketFactor
  case class Volatility(equity: Equity) extends MarketFactor
  case class DaysToMaturity(maturity: LocalDate) extends MarketFactor
  case object RiskFreeRate extends MarketFactor
}

trait MarketFactors {
  def apply(factor: MarketFactor): Option[Double]
}

trait MarketFactorsGenerator {
  def factors: Process[Task, MarketFactors]
}
{% endcodeblock %}

{% codeblock lang:scala %}
sealed trait PricingError

object PricingError {
  case class MissingMarketFactors(factors: NonEmptyList[MarketFactor]) extends PricingError
}

@implicitNotFound(msg = "Can't find pricer for instrument type '${I}'")
trait Pricer[I <: Instrument] {
  def price(i: I)(implicit m: MarketFactors): PricingError \/ Double
}

object Pricer extends PricerImplicits

trait PricerImplicits {

    implicit object EquityPricer extends Pricer[Equity] {
        // ...
    }

    implicit object OptionPricer extends Pricer[EquityOption] {
        // Use Black-Scholes formula to price Option
    }
}
{% endcodeblock %}

### Market Risk Calculator

Portfolio Market Risk depends on market factors forecast. I'm going to construct market factors forecast for one day horizon from historical market data:

{% codeblock lang:scala %}
trait MarketDataModule {

  protected def marketData: MarketData

  sealed trait MarketDataError
  case class MarketDataUnavailable(error: Throwable) extends MarketDataError

  trait MarketData {
    def historicalPrices(equity: Equity, from: LocalDate, to: LocalDate):  MarketDataError \/ Vector[HistoricalPrice]

    def historicalPrice(equity: Equity, date: LocalDate): MarketDataError \/ Option[HistoricalPrice]
  }
}
{% endcodeblock %}

{% codeblock lang:scala %}
trait MarketFactorsModule {

  case class MarketFactorsParameters(riskFreeRate: Double = 0.05,
                                     horizon: Int = 1000)

  protected def oneDayMarketFactors(portfolio: Portfolio, date: LocalDate)(implicit parameters: MarketFactorsParameters): MarketFactorsGenerator

  protected def marketFactors(date: LocalDate)(implicit parameters: MarketFactorsParameters): MarketFactors
}
{% endcodeblock %}

{% codeblock lang:scala %}
trait MarketRiskCalculator {

  type MarketRisk <: MarketRiskLike

  trait MarketRiskLike {
    def VaR(p: Double): Double
    def conditionalVaR(p: Double): Double
  }

  def marketRisk(portfolio: Portfolio, date: LocalDate): MarketRisk
}
{% endcodeblock %}

### Monte Carlo Simulation for Market Risk Calculation

> Monte Carlo simulation performs risk analysis by building models of possible results by substituting a range of values—a probability distribution—for any factor that has inherent uncertainty. It then calculates results over and over, each time using a different set of random values from the probability functions. Depending upon the number of uncertainties and the ranges specified for them, a Monte Carlo simulation could involve thousands or tens of thousands of recalculations before it is complete. Monte Carlo simulation produces distributions of possible outcome values. [...](http://www.palisade.com/risk/monte_carlo_simulation.asp)

#### Abstract Monte Carlo Risk Calculator

I will use [scalaz-stream](/blog/2014/03/09/2014-03-09-scalaz-stream-concurrent-process) for splitting calculation to independent tasks, running them [concurrently](/blog/2014/03/09/2014-03-09-scalaz-stream-concurrent-process) and aggregating results.

{% codeblock lang:scala  Some code is omitted. Full source code on Github -->>> https://github.com/ezhulenev/akka-var-calculation/blob/master/core/src/main/scala/kkalc/service/simulation/MonteCarloMarketRiskCalculator.scala %}
trait PortfolioValueSimulation {
  self: MarketFactorsModule with MonteCarloMarketRiskCalculator =>

  /**
   * Scalaz-Stream Channel that for given market factors generator
   * runs defined number of simulations and produce MarketRisk
   */
  def simulation(portfolio: Portfolio, simulations: Int):
    Channel[Task, MarketFactorsGenerator, Simulations]
}

/**
 * @concurrencyLevel Number of simulation tasks running at the same time
 */
abstract class MonteCarloMarketRiskCalculator(simulations: Int = 1000000,
                                              splitFactor: Int = 10,
                                              concurrencyLevel: Int = 10)
  extends MarketRiskCalculator
     with MarketDataModule
     with MarketFactorsModule
     with PortfolioValueSimulation { calculator =>

  case class Simulations(simulations: Vector[Double])

  class MarketRisk(initialValue: Double, simulations: Simulations) {
    def VaR(p: Double): Double = ...

    def conditionalVaR(p: Double): Double = ...
  }

  private val P = Process

  def marketRisk(portfolio: Portfolio, date: LocalDate): MarketRisk = {

    // Get initial Portfolio value
    implicit val initialFactors = marketFactors(date)
    val initialPortfolioValue =
      PortfolioPricer.price(portfolio).
        fold(err => sys.error(s"Failed price portfolio: $err"), identity)

    // Run portfolio values simulation
    val oneSimulation = simulations / splitFactor
    val simulationChannel = simulation(portfolio, oneSimulation)

    val generator = oneDayMarketFactors(portfolio, date)

    val process = P.
      range(0, splitFactor).
      map(_ => generator).
      concurrently(concurrencyLevel)(simulationChannel).
      runFoldMap(identity)

    // Produce market-risk object from initial value and simulated values
    new MarketRisk(initialPortfolioValue, process.run)
  }
}
{% endcodeblock %}

#### Running simulation on a single node

Here is straightforward PortfolioValueSimulation implementation, that runs simulation tasks in a separate thread pool in a single JVM:

{% codeblock lang:scala %}
trait SingleNodePortfolioValueSimulation extends PortfolioValueSimulation {
  self: MarketFactorsModule with MonteCarloMarketRiskCalculator =>

  def simulation(portfolio: Portfolio, simulations: Int) =
    channel[MarketFactorsGenerator, Simulations] { generator =>

      // calculate portfolio prices for generated market factors
      val process = generator.factors.take(simulations).map {
        implicit factors => PortfolioPricer.price(portfolio).
          fold(err => sys.error(s"Failed to price portfolio: $err"), identity)
      }

      // Fork simulations into thread pool
      Task.fork {
        log.debug(s"Simulate $simulations portfolio values for $portfolio")
        process.runLog.
            map(portfolioValues => Simulations(portfolioValues.toVector))
      }(executor)
  }
}
{% endcodeblock %}

#### Run Market Risk Calculation

{% codeblock lang:scala %}
object SingleNodeMarketRiskCalculation extends App {
  val AMZN = Equity("AMZN")
  val AAPL = Equity("AAPL")
  val IBM = Equity("IBM")
  val GS = Equity("GS")

  // Portfolio evaluation date
  val date = new LocalDate(2014, 1, 3)

  // Options maturity date
  val maturityDate = new LocalDate(2014, 3, 31)

  val portfolio = Portfolio(nels(
    Position(AMZN, 10), Position(AAPL, 20), Position(IBM, 30),
    Position(CallOption(GS, 180, maturityDate), 10)
  ))

  object RiskCalculator
    extends MonteCarloMarketRiskCalculator(10000, 10)
    with SingleNodePortfolioValueSimulation
    with HistoricalMarketFactors with HistoricalMarketData

  val marketRisk = RiskCalculator.marketRisk(portfolio, date)

  println(s"Calculated marker risk; " +
    s"VaR(p = 0.95) = ${marketRisk.VaR(0.95)}, " +
    s"CVaR(p = 0.95) = ${marketRisk.conditionalVaR(0.95)}")
}
{% endcodeblock %}


{% codeblock %}
[simulation-pool-1] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-2] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-0] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-3] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-4] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-5] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-6] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-7] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-8] Simulate 1000 portfolio values for Portfolio( ...
[simulation-pool-9] Simulate 1000 portfolio values for Portfolio( ...

Calculated marker risk in 3492 milliseconds:
        VaR(p = 0.95) = -135.82237858706867
        CVaR(p = 0.95) = -170.1895188516829
{% endcodeblock %}


### Next step

In next part I'm going to scale-out VaR calculation to multiple nodes, and I will be using latest Akka Cluster for it.

## [>>> Go to Part 2](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-2)