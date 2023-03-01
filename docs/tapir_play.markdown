---
layout: default
title: Using Tapir in a Play Framework application
description: THow can we get all the Tapir benefits in a Play Framework application?
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

