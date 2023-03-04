---
layout: default
title: Using Tapir in a Play Framework application
description: How can we get all the Tapir benefits in a Play Framework application?
---

# 🤩 Which benefits?

If you are reading this, you probably already know what Tapir does and why it’s a great library for describing endpoints in a Scala application. But what does it bring specifically to Play Framework applications?

## Types

In a Play `Action` the input type(s) can be defined and known with the controller method signature (parameters and return type) but **the output type(s) doesn’t appear anywhere**.

```scala
def update(bookId: Long): Action[BookDetails] = Action.async(parse.json[BookDetails]) {
  val book: Book = ???
  Future.successful(Ok(Json.toJson(book)))
}
```

In the example above, nothing in the types tells you that the endpoint will always return succesfully with a `Book` instance. And we’re not even asking about the content type of the response.

## Best practices

**Split the business logic from the HTTP logic.** Even though nothing prevents you to apply this pattern in a Play application, nothing forces you to do it either.

With Tapir, the only place where HTTP logic (status code, content type…) can be defined is in the `Endpoint` definition.

## Documentation

If you want to provide OpenAPI definition from your code, you can either use:

- annotations with [Swagger Play](https://github.com/swagger-api/swagger-play) (which is unmaintained for more than 2 years),
- or the more recent [sbt-swagger-play](https://github.com/dwickern/sbt-swagger-play) plugin,
- or even write YAML yourself with [Play Swagger](https://github.com/iheartradio/play-swagger).

All these options share the same issues: your documentation will drift from your code as nothing checks that both are consistent and you have to partially duplicate some piece of information.

> Documentation was my first motive to get into Tapir. Types then come naturally with the documentation. And the encouraged best practices are the icing on the cake!

# 🚀 Exposing an endpoint

Let’s see how to transform a “plain old” Play Framework endpoint to Tapir.

## A classic Play endpoint

Consider the following endpoint defined by an entry in the `routes` file and an `Action` in a controller class:

```
# routes
GET   /api/books/:id      MyController.getBook(id: Long)
```

```scala
// MyController.scala

import play.api.libs.json.Json
import play.api.mvc.{Action, AnyContent, InjectedController}

import javax.inject.Inject
import scala.concurrent.ExecutionContext

class MyController @Inject() (myService: MyService)(implicit ec: ExecutionContext) extends InjectedController {

  def getBook(id: Long): Action[AnyContent] = Action.async {
    myService.findBook(id) // Calling the business logic
      .map { // Mapping business result to HTTP response and code
        case Right(book) =>
          Ok(Json.toJson(book))
        case Left(BookNotFound) =>
          NotFound
        case Left(error: DatabaseError) =>
          InternalServerError(Json.toJson(error))
      }
  }

}
```

```scala
// MyService.scala

import scala.concurrent.Future

class MyService {

  def findBook(id: Long): Future[Either[GetBookError, Book]] = ???

}
```

```scala
// Book.scala

import play.api.libs.json.{Format, Json}

// A business model
case class Book(id: Long, title: String, description: String)

object Book {
  implicit val format: Format[Book] = Json.format[Book]
}
```

```scala
// GetBookError.scala

import play.api.libs.json.{Format, Json}

// A model to represent possible error cases (business and/or technical)
sealed trait GetBookError

case object BookNotFound extends GetBookError

case class DatabaseError(error: String) extends GetBookError

object DatabaseError {
  implicit val format: Format[DatabaseError] = Json.format[DatabaseError]
}
```

Now, let’s transform this with Tapir!

Note that the example uses Play JSON as the JSON library but any other one supported by Tapir (Circe, ZIO Json, json4s…) would work.

You can also notice that we already split the business logic from the HTTP one: the `Action` only calls the business logic which returns a business value (a `Right[Book]`) or an error (`Left[GetBookError]`) and maps it to the corresponding HTTP code and response.

## Describe an endpoint

With Tapir, we need to define a `Endpoint[SEC, IN, ERR, OUT, R]` which is the **complete description of our endpoint**. It will describe the input needed for security (type `SEC`), the regular input(s) (type `IN`), the success output(s) (type `OUT`) and the error output(s) (type `ERR`). The type `R` is used for additional capabilities like streaming or websocket which we won’t use here.

In our example, we have:
- one input: the `id` as a path parameter
- one success output: an instance of `Book` in JSON (HTTP 200)
- two possible error outputs: either we didn’t find a book (`BookNotFound`, no content, HTTP 404), or we got an error with the database (`DatabaseError`, content in JSON, HTTP 500).

This translates to the following definition with Tapir:

```scala
import sttp.model.StatusCode._
import sttp.tapir.json.play._
import sttp.tapir.generic.auto._
import sttp.tapir._

object MyEndpoint {

  val getBookEndpoint: PublicEndpoint[Long, GetBookError, Book, Any] =
    endpoint
      // GET method
      .get
      // Path and input
      .in("api" / "books" / path[Long]("id"))
      // Success output
      .out(jsonBody[Book])
      // Error output
      .errorOut(
        oneOf[GetBookError](
          oneOfVariant[BookNotFound.type](NotFound, emptyOutputAs(BookNotFound)),
          oneOfVariant[DatabaseError](InternalServerError, jsonBody[DatabaseError])
        )
      )

}
```

See how the **types are already helping us**? Just by looking at the type of `getBookEndpoint` we know that we are dealing with an endpoint that will take a `Long` as input and give us back either a `Book` or an error of which we know the possible cases as they are defined in a `sealed trait`.

We can also see with `PublicEndpoint` that this endpoint doesn’t require any authentication. `PublicEndpoint[IN, ERR, OUT, R]` is just a type alias for `Endpoint[Unit, IN, ERR, OUT, R]`.

Note that we defined the endpoint as a “monolithic `val`" here but another strength of Tapir lies in its **composition capabilities**. We could have written the same endpoint with reusable pieces like below in order to reuse the base path of the endpoint as well as the error output mapping.

```scala
val basePath: EndpointInput[Unit] = "api" / "books"

val getBookErrorOutput: EndpointOutput[GetBookError] = oneOf[GetBookError](
  oneOfVariant[BookNotFound.type](NotFound, emptyOutputAs(BookNotFound)),
  oneOfVariant[DatabaseError](InternalServerError, jsonBody[DatabaseError])
)

val getBookEndpoint: PublicEndpoint[Long, GetBookError, Book, Any] =
  endpoint
    .get
    .in(basePath / path[Long]("id"))
    .out(jsonBody[Book])
    .errorOut(getBookErrorOutput)
```

At this stage, our endpoint is well defined but it’s not documented and it doesn’t do anything. Let’s first see how we can bind our endpoint to some actual implementation and expose it as a HTTP resource.

## Keep the logic

**The business logic doesn’t have to change**: we can keep the exact same `findBook` method we used with “classic Play” and we don’t even need to move it elsewhere, it can stay in the `MyService` class.

This is especially true because we already had a method defined as `IN => Future[Either[ERR, OUT]]` which is the “default type” expected by Tapir to bind a `Endpoint[_, IN, ERR, OUT, _]` to some implementation.

Let’s move to the binding!

## Binding the endpoint

Now we are going to bind our endpoint to some implementation. It’s as easy as using the method `serverLogic(...)` (or one of its variants) on the `Endpoint` defined previously.

```scala
import sttp.tapir.server.ServerEndpoint

import javax.inject.{Inject, Singleton}
import scala.concurrent.Future

@Singleton
class TapirRouter @Inject() (myService: MyService) {

  // Binding
  private val getBookServerEndpoint: ServerEndpoint[Any, Future] =
    MyEndpoint.getBookEndpoint.serverLogic(myService.findBook)

}
```

Here we inject the `MyService` class and declare that our `getBookEndpoint` is actually implemented with the `findBook` method.

This gives us a `ServerEndpoint` which has nothing related to Play. If we were in a Akka HTTP, http4s or ZIO-HTTP application, we would have written pretty much the same code.

Note that there are several variants depending on how the business method is defined like `serverLogicPure` which accepts a `IN => Either[ERR, OUT]` business method, that is without a `Future`.

## Letting Play know about the Routes

It’s now time to get back to Play and somehow integrate what we did with Tapir so far.

With Play, we are used to defined routes using the `routes` file. But did you know there’s another way around? It’s called the **SIRD Router** (“String Interpolation Routing DSL”). It’s a Scala DSL that allows to define routes with Scala code. You might have already used it in some tests when you bring up a mock server. Learn more about it here.

This SIRD Router is the way to bridge the gap between Play and Tapir!

Let’s complete the `TapirRouter` class we wrote in previous step and make it extends `play.api.routing.SimpleRouter`. Then, let’s ask Tapir to generate a Play `Routes` for us:

```scala
import akka.stream.Materializer
import play.api.routing.Router.Routes
import play.api.routing.SimpleRouter
import sttp.tapir.server.ServerEndpoint
import sttp.tapir.server.play.{PlayServerInterpreter, PlayServerOptions}

import javax.inject.{Inject, Singleton}
import scala.concurrent.{ExecutionContext, Future}

@Singleton
class TapirRouter @Inject() (myService: MyService)(implicit ec: ExecutionContext, mat: Materializer) extends SimpleRouter {

  private val getBookServerEndpoint: ServerEndpoint[Any, Future] =
    MyEndpoint.getBookEndpoint.serverLogic(myService.findBook)

  // Tapir interpreter to generate Play Routes
  private val interpreter = PlayServerInterpreter(PlayServerOptions.default)

  val getBookRoute: Routes = interpreter.toRoutes(getBookServerEndpoint)

  override def routes: Routes = getBookRoute // .orElse(anotherRoute)

}
```

Notice how we instantiated a `PlayServerInterpreter` which is the Tapir class responsible for transforming a `ServerEndpoint` into something that Play can work with: a `Routes`.

The last piece of the puzzle is to “register” our `TapirRouter`. This can be done either with a configuration key `play.http.router` in `application.conf` or within the good old `routes` file itself. The second option has the advantage that you can still write some plain old Play `Action`s and declare them from the `routes` file.

```
# Classic Play route
#GET   /api/books/:id      MyController.getBook(id: Long)

# Tapir router
->     /                   TapirRouter
```

Don’t forget to remove the old route declaration!

You can combine `Routes` by using `firstRoute.orElse(secondRoute)` method when you have several endpoints. This means you can also combine several `Router`s classes by using `router.routes.orElse(anotherRouter.routes)`. The order of declaration is the order of precedence if some routes conflict with each other.

**🎉 And that’s it! We have a Play application where the routes are defined using Tapir.**

It’s important to note that Play (actually Akka HTTP or Netty) is still the runtime HTTP server used. Tapir only brings a way to define the routes and some encoders/decoders.

# 📝 What about the documentation?

But wait… we said one of the benefits of Tapir was documentation. There’s no documentation for now!

## Enrich the endpoint

Let’s enrich our endpoint definition with some descriptions for the input and output:

```scala
val getBookEndpoint: PublicEndpoint[Long, GetBookError, Book, Any] =
  endpoint.get
    .in("api" / "books" / path[Long]("id").description("Book identifier"))
    .out(jsonBody[Book].description("The book if found"))
    .errorOut(
      oneOf[GetBookError](
        oneOfVariant[BookNotFound.type](NotFound, emptyOutputAs(BookNotFound).description("If no book for the identifier")),
        oneOfVariant[DatabaseError](InternalServerError, jsonBody[DatabaseError])
      )
    )
```

Note the use of `.description(...)` method on both the inputs and outputs. There are other methods for defining everything that we can define with OpenAPI like `.example(...)` or `.default(...)`. That is not mandatory to generate OpenAPI documentation though: our endpoint as it was before could still have generated a nice documentation.

## Generating and exposing the OpenAPI documentation

The next step is to generate the OpenAPI documentation for a list of `Endpoint`s that we want to document.

Tapir comes with a **built-in support for SwaggerUI (and Redoc)** but you can also generate the OpenAPI YAML only and use it in a custom way. Let’s look at the most straightforward way to do it:

```scala
import sttp.apispec.openapi.Info
import sttp.tapir.swagger.SwaggerUIOptions
import sttp.tapir.swagger.bundle.SwaggerInterpreter

import scala.concurrent.Future

object OpenApi {

  val swaggerEndpoints = SwaggerInterpreter(swaggerUIOptions = SwaggerUIOptions.default)
    .fromEndpoints[Future](
      // All the endpoints to be documented
      List(MyEndpoint.getBookEndpoint),
      // OpenAPI additional info
      Info("My awesome application", "1.0.0")
    )

}
```

This gives us a list of `ServerEndpoint` which corresponds to the SwaggerUI bundle (HTML/CSS/JS) and the OpenAPI specification YAML for our endpoints.

By default (see `SwaggerUIOptions`) the documentation is exposed under _/docs_.

Note that you can hide some endpoints from the documentation by not declaring them here. Or you can generate multiple documentations with different subsets of endpoints at different paths.

Then, it’s just a matter of exposing these documentation endpoints as any other endpoint in the router:

```scala
import akka.stream.Materializer
import play.api.routing.Router.Routes
import play.api.routing.SimpleRouter
import sttp.tapir.server.ServerEndpoint
import sttp.tapir.server.play.{PlayServerInterpreter, PlayServerOptions}

import javax.inject.{Inject, Singleton}
import scala.concurrent.{ExecutionContext, Future}

@Singleton
class TapirRouter @Inject() (myService: MyService)(implicit ec: ExecutionContext, mat: Materializer) extends SimpleRouter {

  private val getBookServerEndpoint: ServerEndpoint[Any, Future] =
    MyEndpoint.getBookEndpoint.serverLogic(myService.findBook)

  // Tapir interpreter to generate Play Routes
  private val interpreter = PlayServerInterpreter(PlayServerOptions.default)

  val getBookRoute: Routes = interpreter.toRoutes(getBookServerEndpoint)

  val documentationRoute: Routes = interpreter.toRoutes(OpenApi.swaggerEndpoints)

  override def routes: Routes = getBookRoute.orElse(documentationRoute)

}
```

# 🏁 Conclusion

Even though Tapir is a new way of writing APIs in Scala, we’ve seen that it integrates nicely with “good old” frameworks like Play.

**The vast majority of what you can do with Play, you can do it with Tapir as well.**

And what you cannot do today might be solved in the next release of Tapir! The contributors behind Tapir are very reactive if you open an issue on [Github](https://github.com/softwaremill/tapir).

ℹ️ You can find a sample project with more examples of integration between Play Framework and Tapir at [https://github.com/gaeljw/tapir-play-sample](https://github.com/gaeljw/tapir-play-sample).

## Links

- Tapir Github: [https://github.com/softwaremill/tapir](https://github.com/softwaremill/tapir)
- Tapir documentation: [https://tapir.softwaremill.com/en/latest/index.html](https://tapir.softwaremill.com/en/latest/index.html)
- Tapir Play sample project: [https://github.com/gaeljw/tapir-play-sample](https://github.com/gaeljw/tapir-play-sample)

_Note: the code presented here has been written with Play Framework 2.8.16 and Tapir 1.0.0._
