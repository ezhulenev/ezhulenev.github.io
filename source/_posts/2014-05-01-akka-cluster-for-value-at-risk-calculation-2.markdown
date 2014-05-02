---
layout: post
title: "Akka Cluster for Value at Risk calculation (Part 2/2)"
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


## [Part 2/2] Scale-out VaR calculation to multiple nodes

Go to [Part 1](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-1) where Value at Risk calculator defined.

<!-- more -->

### Akka Cluster

Akka is amazing library for Actors abstraction for Scala and Java.

> Actors are location transparent and distributable by design. This means that you can write your application without hardcoding how it will be deployed and distributed, and then later just configure your actor system against a certain topology [...](http://doc.akka.io/docs/akka/snapshot/general/remoting.html)

Akka Cluster provides a fault-tolerant decentralized peer-to-peer based cluster membership service with no single point of failure or single point of bottleneck. It does this using gossip protocols and an automatic failure detector.

It means that it's very easy to distribute portfolio simulations to multiple nodes.


### Messages

Messages that are passed around between backend & calculator nodes:

{% codeblock lang:scala %}
object messages {
  case object WakeUp

  case object RegisterBackend

  case class RunSimulation(positions: List[Position],
                           simulations: Int,
                           generator: MarketFactorsGenerator)

  case class SimulationResponse(v: Vector[Double])
}
{% endcodeblock %}


### Simulation Backend Node

First type of node in a cluster is SimulationNode that is going to run all 'heavy' portfolio price simulations.

After joining the cluster it subscribes for all `MemberUp` messages, and when new node with role `calculator` joins the cluster, it register itself as available backend.

{% codeblock lang:scala %}
class PortfolioValueSimulationBackend extends Actor with ActorLogging {

  private val cluster = Cluster(context.system)

  override def preStart(): Unit = cluster.subscribe(self, classOf[MemberUp])

  override def postStop(): Unit = cluster.unsubscribe(self)

  def receive = {
    case RunSimulation(portfolio, simulations, generator) =>
      runSimulation(
        Portfolio(nel(portfolio.head, portfolio.tail)),
        simulations, generator
      ) pipeTo sender()

    case state: CurrentClusterState =>
      state.members.filter(_.status == MemberStatus.Up) foreach register

    case MemberUp(member) =>
      register(member)
  }

  private def register(member: Member): Unit =
    if (member.hasRole("calculator")) {
      val memberRoot = RootActorPath(member.address)
      val backendManager =
        context.actorSelection(memberRoot / "user" / "backendManager")
      backendManager ! RegisterBackend
    }

  private def runSimulation(portfolio: Portfolio,
        simulations: Int,
        generator: MarketFactorsGenerator): Future[SimulationResponse] = {

    val process = generator.factors.take(simulations).map {
      implicit factors =>
        PortfolioPricer.price(portfolio).
            fold(err => sys.error(s"Failure: $err"), identity)
    }

    val task = Task.fork {
      process.runLog.map(portfolioValues => portfolioValues.toVector)
    }(executor)

    val p = Promise[SimulationResponse]()

    task.runAsync {
      case -\/(err) => p.failure(err)
      case \/-(result) => p.success(SimulationResponse(result))
    }

    p.future
  }
}
{% endcodeblock %}


### Calculator Node

Calculator node joins the cluster, and receive `RegisterBackend` messages from simulation nodes. It keeps track of all available simulation backend nodes, and when gets a request to calculate market risk,
it splits this request into multiple simulation tasks and send them to all available simulation backends.

##### Backend Nodes Manager

{% codeblock lang:scala %}
class BackendNodesManager extends Actor with ActorLogging {

  private[this] val backendNodes = ListBuffer.empty[ActorRef]
  private[this] var jobCounter = 0

  private[this] implicit val timeout = Timeout(10.seconds)

  override def receive: Receive = {
    case WakeUp => log.info("Wake up backend nodes manager")

    case run: RunSimulation =>
      jobCounter += 1
      val backendN = jobCounter % backendNodes.size
      log.debug(s"Pass simulation request to backend: $backendN")
      backendNodes(backendN) ? run pipeTo sender()

    case RegisterBackend if !backendNodes.contains(sender()) =>
      context watch sender()
      backendNodes += sender()
      log.debug(s"Added new backend. "+
                s"Total: ${backendNodes.size}. Node: [${sender()}}]")

    case Terminated(backEnd) if backendNodes.contains(backEnd) =>
      backendNodes -= sender()
      log.debug(s"Removed terminated backend. "+
                s"Total: ${backendNodes.size}. "+
                s"Terminated node: [${sender()}}]")
  }
}
{% endcodeblock %}


##### Cluster Portfolio Value Simulation

Simulation channel constructed by `asking` backend simulation actors to run simulation and converting `scala.concurrent.Future` to `scalaz.concurrent.Task`. Concurrency management and split factor is defined
in abstract monte carlo risk calculator described in [Part 1](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-1)

{% codeblock lang:scala %}
trait ClusterPortfolioValueSimulation extends PortfolioValueSimulation {
  self: MarketFactorsModule with MonteCarloMarketRiskCalculator =>

  def systemName: String

  def systemConfig: Config

  private[this] lazy val system = ActorSystem(systemName, systemConfig)

  private[this] lazy val backendManager =
    system.actorOf(Props[BackendNodesManager], "backendManager")

  def simulation(portfolio: Portfolio, simulations: Int) =
    channel[MarketFactorsGenerator, Simulations] { generator =>
      Task.async[Simulations](cb => {
        (backendManager ? RunSimulation(...)).onComplete {
          case Failure(err)                   => cb(-\/(err))
          case Success(SimulationResponse(s)) => cb(\/-(Simulations(s)))
          case Success(u)                     =>
            cb(-\/(new IllegalStateException(s"Unknown response: $u")))
        }
      })
  }
}
{% endcodeblock %}

### Run Calculation in Cluster

In this example I'm going to run all nodes in a single JVM for simplicity. Distributed deployment is only Akka configuration issue, and it doesn't affect the code at all.

I'm starting three `simulation backend` nodes in a cluster, and later join them with `calculator` node, and submit risk calculation task.

{% codeblock lang:scala %}
object ClusterMarketRiskCalculation extends App {
  val SystemName = "ClusterMarketRiskCalculation"

  val simulationConfig = ConfigFactory.parseResources("simulation-node.conf")
  val calculatorConfig = ConfigFactory.parseResources("calculator-node.conf")

  // Start 3 simulation nodes
  val system1  = ActorSystem(SystemName, simulationConfig)
  val joinAddress = Cluster(system1).selfAddress
  Cluster(system1).join(joinAddress)
  system1.actorOf(Props[PortfolioValueSimulationBackend], "simulationBackend")

  val system2  = ActorSystem(SystemName, simulationConfig)
  Cluster(system2).join(joinAddress)
  system2.actorOf(Props[PortfolioValueSimulationBackend], "simulationBackend")

  val system3  = ActorSystem(SystemName, simulationConfig)
  Cluster(system3).join(joinAddress)
  system3.actorOf(Props[PortfolioValueSimulationBackend], "simulationBackend")

  // Start Cluster Risk Calculator node
  object RiskCalculator
    extends MonteCarloMarketRiskCalculator(10000, 10)
    with ClusterPortfolioValueSimulation
    with HistoricalMarketFactors with HistoricalMarketData {

    val systemName = SystemName
    val systemConfig = calculatorConfig
  }

  RiskCalculator.join(joinAddress)

  // Let's cluster state some time to converge
  Thread.sleep(2000)

  // Run VaR calculation

  val AMZN = Equity("AMZN")
  val AAPL = Equity("AAPL")
  val IBM = Equity("IBM")
  val GS = Equity("GS")

  // Portfolio evaluation date
  val date = new LocalDate(2014, 1, 3)

  // Options maturity date
  val maturityDate = new LocalDate(2014, 3, 31)

  val portfolio = Portfolio(nels(
    Position(AMZN, 10),
    Position(AAPL, 20),
    Position(IBM, 30),
    Position(CallOption(GS, 180, maturityDate), 10)
  ))

  val start = System.currentTimeMillis()
  val marketRisk = RiskCalculator.marketRisk(portfolio, date)
  val end = System.currentTimeMillis()

  println(s"Calculated marker risk in ${end - start} milliseconds; " +
    s"VaR(p = 0.95) = ${marketRisk.VaR(0.95)}, " +
    s"CVaR(p = 0.95) = ${marketRisk.conditionalVaR(0.95)}")

  // Shutdown actor systems
  system1.shutdown()
  system2.shutdown()
  system3.shutdown()
  RiskCalculator.shutdown()

  // and application
  System.exit(0)
}
{% endcodeblock %}


### Conclusion

As I showed in this post moving calculations to a cluster can be easy and fun task with Akka Cluster.

I use scalaz-stream for abstracting over `effectful functions` with `scalaz.stream.Channel`, I guess it maybe be overengineering in this particular case,
but it allows to completely hide implementation details and take control concurrency in very abstract way.
And scalaz-stream is very nice and super powerful library, I strongly encourage you to take a look on it.

##### Links

1. Akka http://akka.io/ and http://doc.akka.io/docs/akka/2.3.2/scala/index-network.html
2. Scalaz-stream https://github.com/scalaz/scalaz-stream
3. Value at Risk http://en.wikipedia.org/wiki/Value_at_risk

####  Check source code on [GitHub](https://github.com/ezhulenev/akka-var-calculation)

## [<<< Go to Part 1](/blog/2014/05/01/akka-cluster-for-value-at-risk-calculation-1)
