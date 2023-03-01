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
