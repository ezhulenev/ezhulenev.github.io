---
layout: post
title: "Seamless migration from monolithic application to Finagle services (Part 2/2)"
date: 2014-04-09 22:03:45 -0400
comments: true
categories: [scala, finagle, micro-services, zookeper, cloud]
keywords: scala, finagle, micro-services, zookeper, cloud
---

### Synopsis

> The code & sample app can be found on [Github](https://github.com/ezhulenev/finagled-movie-db)

Distributed micro-services architecture is hot trend right now, it's widely adopted by [Twitter](https://blog.twitter.com/2011/finagle-a-protocol-agnostic-rpc-system), [LinkedIn](https://engineering.linkedin.com/architecture/restli-restful-service-architecture-scale) and [Gilt](http://www.slideshare.net/LappleApple/gilt-from-monolith-ruby-app-to-micro-service-scala-service-architecture).
However it can be difficult if data model is already defined, and services accessed via existing API throughout all your code. I’ll show how it's possible to split monoliths app into standalone services built with Finagle and SBinary for custom communication protocol.

I'm going to show it on example of small Fancy Movie Database application.

## [Part 2/2] Spit application to distributed services

Go to [Part 1](/blog/2014/04/09/seamless-migration-from-monolithic-application-to-finagle-services-1) where fancy movie database application defined.

### What is Finagle

Finagle is a protocol-agnostic, asynchronous RPC system for the JVM that makes it easy to build robust clients and servers in Java, Scala, or any JVM-hosted language.

<!-- more -->

###### Finagle provides a robust implementation of:
- connection pools, with throttling to avoid TCP connection churn;
- failure detectors, to identify slow or crashed hosts;
- failover strategies, to direct traffic away from unhealthy hosts;
- load-balancers, including “least-connections” and other strategies; and

You can read more about finagle on [official web site](http://twitter.github.io/finagle/)

#### What's wrong with finagle

Finagle is protocol agnostic system, and can work independently of underlying protocol, however suggested protocol is Thrift, and tooling support is built around Thrift (code generators, etc).
One biggest drawbacks of Thrift, is that it's required to define model and services using interface definition language (IDL).
However if model and services already defined (as in this example), it can be painful to migrate well-typed scala model to IDL.

In this case we can use protocol-agnostic property of Finagle and write out own binary protocol for existing scala model.

### SBinary

[SBinary](https://github.com/harrah/sbinary) is a library for describing binary protocols, in the form of mappings between Scala types and binary formats. It can be used as a robust serialization mechanism for Scala objects or a way of dealing with existing binary formats found in the wild.

> Great [Introduction to SBinary](https://code.google.com/p/sbinary/wiki/IntroductionToSBinary) article


### Binary format for data model

First we need a way to read/write data model from binary representation:

{% codeblock lang:scala %}
trait ModelProtocol {

  import sbinary.DefaultProtocol._
  import sbinary.Operations._
  import sbinary._

  implicit object genreFormat extends Format[Genre] {
    import Genre._

    override def reads(in: Input): Genre = read[Byte](in) match {
      case 0 => Action
      case 1 => Adventure
      ....
    }

    override def writes(out: Output, value: Genre) = {
      val genreCode: Byte = value match {
        case Action      => 0
        case Adventure   => 1
        ....
      }
      write[Byte](out, genreCode)
    }
  }

  implicit object personFormat extends Format[Person] {
    override def reads(in: Input) =
      Person(read[String](in), read[String](in), read[LocalDate](in))

    override def writes(out: Output, person: Person) {
      write[String](out, person.firstName)
      write[String](out, person.secondName)
      write[LocalDate](out, person.born)
    }
  }

   // ... much more on Github

}
{% endcodeblock %}


### Binary format for Request/Response

Nest step, is to build request-response commands that going to be passed between server and client:

{% codeblock lang:scala %}
object ServiceProtocol extends ModelProtocol {

  sealed trait FmdbReq
  case object GetPeople extends FmdbReq
  case class GetMovies(year: Option[Int], genre: Option[Genre], person: Option[Person]) extends FmdbReq

  sealed trait FmdbRep
  case class GotPeople(people: Vector[Person]) extends FmdbRep
  case class GotMovies(movies: Vector[Movie]) extends FmdbRep

  implicit object requestFormat extends Format[FmdbReq] {
    override def reads(in: Input) = read[Byte](in) match {
      case 0 =>
           GetPeople

      case 1 =>
           GetMovies(read[Option[Int]](in),
                     read[Option[Genre]](in),
                     read[Option[Person]](in))
    }

    override def writes(out: Output, req: FmdbReq) {
      req match {
        case GetPeople =>
          write[Byte](out, 0)

        case GetMovies(year, genre, person) =>
           write[Byte](out, 1)
           write[Option[Int]](out, year)
           write[Option[Genre]](out, genre)
           write[Option[Person]](out, person)
      }
    }
  }

  // ... the same for Response, see more on Github
}
{% endcodeblock %}

### Combine it all together

Now we need to combine together binary protocol defined earlier with Finagle channel pipelines, and create Client/Server builders.

For sure we want to update this binary protocol at some later point, adding new commands and updating application model, and to be it still safe.
For this reason I'm wrapping each message into `versioned envelop`, however I'm not going to describe it in this post, full code is available on Github.

{% codeblock lang:scala %}
object Fmdb {
  import ServiceProtocol._
  import envelopeCodec._

  private[this] val ProtocolVersion: Long = 1l
  private[this] val ReqDecoder =
     versionCheckingEnvelopeToContentDecoder[FmdbReq](ProtocolVersion)
  private[this] val RepDecoder =
     versionCheckingEnvelopeToContentDecoder[FmdbRep](ProtocolVersion)
  private[this] val ReqEncoder = typeSafeEncoder[FmdbReq]
  private[this] val RepEncoder = typeSafeEncoder[FmdbRep]

  private[this] object FmdbServerPipeline extends ChannelPipelineFactory {
    def getPipeline = {
      val pipeline = Channels.pipeline()
      pipeline.addLast("envDecoder", new EnvelopeDecoder)
      pipeline.addLast("reqDecoder", ReqDecoder)
      pipeline.addLast("envEncoder", new EnvelopeEncoder(ProtocolVersion))
      pipeline.addLast("repEncoder", RepEncoder)
      pipeline
    }
  }

  private[this] object FmdbClientPipeline extends ChannelPipelineFactory {
    def getPipeline = {
      val pipeline = Channels.pipeline()
      pipeline.addLast("envEncode", new EnvelopeEncoder(ProtocolVersion))
      pipeline.addLast("reqEncode", ReqEncoder)
      pipeline.addLast("envDecode", new EnvelopeDecoder)
      pipeline.addLast("repDecode", RepDecoder)
      pipeline
    }
  }

  private[this] object FmdbClientTransporter extends Netty3Transporter[FmdbReq, FmdbRep](
    "fmdbClientTransporter", FmdbClientPipeline
  )

  private[distributed] object Client extends DefaultClient[FmdbReq, FmdbRep](
    "fmdbClient", endpointer = {
      val bridge =
        Bridge[FmdbReq, FmdbRep, FmdbReq, FmdbRep](FmdbClientTransporter, new SerialClientDispatcher(_))
      (addr, stats) => bridge(addr, stats)
    }
  )

  private[this] object FmdbListener extends Netty3Listener[FmdbRep, FmdbReq](
    "fmdbListener", FmdbServerPipeline
  )

  protected[distributed] object Server extends DefaultServer[FmdbReq, FmdbRep, FmdbRep, FmdbReq](
    "fmdbServer", FmdbListener, new SerialServerDispatcher(_, _)
  )
}
{% endcodeblock %}


### Services implementation

Now we can implement services defined in first part, that will perform network call to Finagle server, instead of running computations/data-fetching locally:

{% codeblock lang:scala %}
trait FmdbServerConfig {
  def serverAddress: SocketAddress

  import ServiceProtocol._

  private[this] val retry = new RetryingFilter[FmdbReq, FmdbRep](
    retryPolicy = RetryPolicy.tries(3),
    timer = DefaultTimer.twitter
  )

  private[this] val timeout = new TimeoutFilter[FmdbReq, FmdbRep](
    timeout = Duration.fromSeconds(10),
    timer = DefaultTimer.twitter
  )

  protected lazy val client =
    retry      andThen
    timeout    andThen
    Fmdb.Client.newService(Name.bound(serverAddress), "fmdbClient")
}

trait PeopleServiceImpl extends PeopleService with FmdbServerConfig {
  import ServiceProtocol._

  def people(): Future[Vector[Person]] = client(GetPeople).map {
    case GotPeople(people) => people
    case err => sys.error(s"Unexpected server response: $err")
  }.toScala

}

// ... more on Github

{% endcodeblock %}

I'm not going to describe hot convert Twitter Future to Scala Future, it's all available on [Github](https://github.com/ezhulenev/finagled-movie-db).

### Let's run it!

Let's find all movies with Leonardo DiCaprio, as we did in first part. However now example application will be a client that will be sending requests via network to movies service server.

{% codeblock lang:scala %}
object DistributedExample extends App {

  val address = new InetSocketAddress(10000)

  // Lets start server
  val server = FmdbServer.serve(address)

  // Lets create services
  object Services extends PeopleServiceImpl with MoviesServiceImpl {
    val serverAddress: SocketAddress = address
  }

  // Lets' get all movies for Leonardo DiCaprio
  val leo = Await.
    result(Services.people(), 1.second).find(_.firstName == "Leonardo").get
  val leoMovies = Await.
    result(Services.movies(leo), 1.second)

  println(s"Movies with ${leo.firstName} ${leo.secondName}:")
  leoMovies.map(m => s" - ${m.title}, ${m.year}").foreach(println)

  // Shutdown
  server.close()
  System.exit(0)
}
{% endcodeblock %}


##### Output

{% codeblock %}
Apr 09, 2014 2:02:18 PM com.twitter.finagle.Init$ apply
INFO: Finagle version 6.13.1 (rev=12bb3f3f5004109a4c2b981091a327b6ba2e7a6a) built at 20140324-225705
Movies with Leonardo DiCaprio:
 - Django Unchained, 2012
{% endcodeblock %}

### Result

As you can see final application in Part 1 is pretty the same is in Part 2.
But with Finagle server-side part of application can be scaled independently, and completely transparent to the client.

Finagle has amazing cluster discovery support, built on top of Zookeeper and ServerGroups, and it's perfect choice for cloud environment.



## [<<< Go to Part 1](/blog/2014/04/09/seamless-migration-from-monolithic-application-to-finagle-services-1)
