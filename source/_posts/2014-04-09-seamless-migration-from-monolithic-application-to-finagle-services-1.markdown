---
layout: post
title: "Seamless migration from monolithic application to Finagle services (Part 1/2)"
date: 2014-04-09 22:03:45 -0400
comments: true
categories: [scala, finagle, micro-services, cloud]
keywords: scala, finagle, micro-services, cloud
---

### Synopsis

> The code & sample app can be found on [Github](https://github.com/ezhulenev/finagled-movie-db)

Distributed micro-services architecture is hot trend right now, it's widely adopted by [Twitter](https://blog.twitter.com/2011/finagle-a-protocol-agnostic-rpc-system), [LinkedIn](https://engineering.linkedin.com/architecture/restli-restful-service-architecture-scale) and [Gilt](http://www.slideshare.net/LappleApple/gilt-from-monolith-ruby-app-to-micro-service-scala-service-architecture).
However it can be difficult if data model is already defined, and services accessed via existing API throughout all your code. I’ll show how it's possible to split monoliths app into standalone services built with Finagle and SBinary for custom communication protocol.

I'm going to show it on example of small Fancy Movie Database application.

## [Part 1] Fancy Movie Database application

Go to [Part 2](/blog/2014/04/09/seamless-migration-from-monolithic-application-to-finagle-services-2) where fancy movie database application is divided into server and client using Finagle.

<!-- more -->

### Data model

Let's start with application data model definition:

{% codeblock lang:scala %}
sealed trait Genre

object Genre {
  case object Action      extends Genre
  case object Adventure   extends Genre
  case object Animation   extends Genre
  case object Biography   extends Genre
  case object Comedy      extends Genre
  case object Crime       extends Genre
  case object Documentary extends Genre
  case object Drama       extends Genre
  case object Thriller    extends Genre
  case object Western     extends Genre
}

case class Person(firstName: String, secondName: String, born: LocalDate)

case class Cast(person: Person, as: String)

case class Movie(title: String,
                 genre: Genre,
                 year: Int,
                 directedBy: Person,
                 cast: Vector[Cast])

{% endcodeblock %}


### Services definition

And services that we want to provide:

{% codeblock lang:scala %}
trait PeopleService {
  def people(): Future[Vector[Person]]
}

trait MoviesService {
  def movies(): Future[Vector[Movie]]

  def movies(year: Int): Future[Vector[Movie]]

  def movies(genre: Genre): Future[Vector[Movie]]

  def movies(person: Person): Future[Vector[Movie]]
}
{% endcodeblock %}


### Mock data access layer

In real application we probably would have SQL database or some other type of storage with movies data.
Let's assume that this storage is `blocking resource`, and quite expensive to access.
However for this example application I will use small in-memory mock implementation of access layer with only two movies directed by [Quentin Tarantino](http://www.imdb.com/name/nm0000233/): [Django Unchained](http://www.imdb.com/title/tt1853728/?ref_=nm_flmg_dr_3), and [Inglourious Basterds](http://www.imdb.com/title/tt0361748/?ref_=nm_flmg_dr_4)

> Source code for access layer on Github: [MovieDatabaseAccess.scala](https://github.com/ezhulenev/finagled-movie-db/blob/master/core/src/main/scala/com/fmdb/MovieDatabaseAccess.scala)


### Let's be reactive

I hope we all agree that applications needs to be `reactive`, that's why I'm wrapping `blocking` calls into `scala.concurrent.future`.

Straightforward services layer implementation:

{% codeblock lang:scala %}
package object monolithic {
trait PeopleServiceImpl extends PeopleService {
  override def people() =
       future { MovieDatabaseAccess.people() }
}

trait MoviesServiceImpl extends MoviesService {
  override def movies(person: Person) =
       future { MovieDatabaseAccess.movies(person) }

  override def movies(genre: Genre) =
       future { MovieDatabaseAccess.movies(genre) }

  override def movies(year: Int) =
       future { MovieDatabaseAccess.movies(year) }

  override def movies() =
       future { MovieDatabaseAccess.movies() }
}
{% endcodeblock %}


### Let's combine it together

After model and services are defined, and basic implementation is provided we can build simple application for querying Fancy Movies Database.

Let's find all movies with Leonardo DiCaprio:

{% codeblock lang:scala %}
object MonolithicExample extends App {

  // Lets create services
  object Services extends PeopleServiceImpl with MoviesServiceImpl

  // Lets' get all movies for Leonardo DiCaprio
  val leo = Await.
    result(Services.people(), 1.second).find(_.firstName == "Leonardo").get

  val leoMovies = Await.
    result(Services.movies(leo), 1.second)

  println(s"Movies with ${leo.firstName} ${leo.secondName}:")
  leoMovies.map(m => s" - ${m.title}, ${m.year}").foreach(println)

  // Shutdown
  System.exit(0)
}
{% endcodeblock %}


##### Output

{% codeblock %}
Movies with Leonardo DiCaprio:
 - Django Unchained, 2012
{% endcodeblock %}



### Next step

In next part I'm going to divide this application into server and client preserving existing API, such that migration from monolithic application to distributed is completely seamless for `service clients`.

## [>>> Go to Part 2](/blog/2014/04/09/seamless-migration-from-monolithic-application-to-finagle-services-2)