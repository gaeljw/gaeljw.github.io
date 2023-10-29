---
layout: default
title: Custom logic from Accept header with Tapir
description: How to use the Accept header to implement custom logic in a Tapir endpoint?
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

# 🎯 Goal

Imagine a endpoint in which you need to apply custom logic based on the `Accept` header of the requests.

For instance, the endpoint might return the data in different formats like JSON or plain text.

# 😐 The naive approach

The naive approach would be to implement an `Endpoint` that:
- reads the `Accept` header as an input parameter
- outputs a `String` mapped to multiple formats

Something like this:
```scala
import sttp.tapir.*

val myEndpoint: PublicEndpoint[String, Unit, String, Any] = endpoint
    .get
    .in("myRoute")
    .in(
      // Hide the header in OpenApi doc as it's already automatically provided due to the response Content-Type choices
      header[String]("Accept").schema(_.hidden(true))
    )
    .out(
      oneOfBody(
        // Order is important: 1st one will be the default if no Accept header provided
        // It should match server implementation behavior as well
        stringBodyAnyFormat(Codec.string.format(CodecFormat.Json()), StandardCharsets.UTF_8),
        stringBodyAnyFormat(Codec.string.format(CodecFormat.TextPlain()), StandardCharsets.UTF_8)
      )
    )
```

And then implement the business logic like this:
```scala
import sttp.model.MediaType

def businessLogic(acceptHeader: String): Future[Either[Unit, String]] = {
    // Extract the MediaType from the Accept header
    val requestedMediaType: MediaType = MediaType.unsafeParse(acceptHeader)
    // Implement real logic
    val str: String = if (requestedMediaType == MediaType.TextPlain) {
        "some plain text"
    } else {
        "some json"
    }
    Future.successful(Right(str))
}
```

👌 **This works fine in simple cases** where the `Accept` request header contains a single value like `application/json` or `text/plain`.

It also sets automatically the `Content-Type` header in the response correctly as Tapir does it automatically from the `Accept` request header.

💥 **But... it doesn't work when you start using real-life complex `Accept` headers** value like `application/json,text/plain;q=0.9` which define several accepted content types with some preference rules.

# 🤯 The right approach

In order to make it work in complex cases, we only need to tweak a bit the previous code.

The first thing is to **read the `Accept` header as a list of types sorted by preference** rather than a simple `String`.

To do so, Tapir gives us a nice helper:

```scala
import sttp.model.ContentTypeRange

// Accept header is hidden in SwaggerUI because response "Content-type" can be chosen in the UI and sets it implicitly
// This is automatic when using extractFromRequest, otherwise we could have used .schema(_.copy(hidden = true))
private val acceptHeader: EndpointInput[Seq[ContentTypeRange]] = 
    extractFromRequest(_.acceptsContentTypes)
    // Extract the Right part of acceptsContentTypes
    .mapDecode(accepts => DecodeResult.Value(accepts.getOrElse(Seq.empty)))(Right(_))
```

That we can then use in the endpoint definition as a replacement for the raw header we used previously:

```diff
-    .in(header[String]("Accept").schema(_.hidden(true)))
+    .in(acceptHeader)
```

For a complete result of:
```scala
val myEndpoint: PublicEndpoint[String, Unit, String, Any] = endpoint
    .get
    .in("myRoute")
    .in(acceptHeader)
    .out(
      oneOfBody(
        stringBodyAnyFormat(Codec.string.format(CodecFormat.Json()), StandardCharsets.UTF_8),
        stringBodyAnyFormat(Codec.string.format(CodecFormat.TextPlain()), StandardCharsets.UTF_8)
      )
    )
```

Then, in the business logic, we can also use a helper from Tapir to implement some logic based on the accepted types:

```scala
import sttp.model.{ContentTypeRange, MediaType}

// Note: Tapir already does a check automatically against the possible output types defined in the endpoint
// This method purpose is to extract the type to use
// Theoretically this will always lead a Some() if supportedTypes is not empty
private def extractBestType(accepts: Seq[ContentTypeRange], supportedTypes: Seq[MediaType]): Option[MediaType] = {
    if (accepts.isEmpty) {
      supportedTypes.headOption
    } else {
      MediaType.bestMatch(supportedTypes, accepts)
    }
}

def businessLogic(acceptHeader: Seq[ContentTypeRange]): Future[Either[Unit, String]] = {
    extractBestType(acceptHeader, Seq(MediaType.ApplicationJson, MediaType.TextPlain)) match {
        case Some(requestedMediaType) =>
            // Implement real logic
            val str: String = if (requestedMediaType == MediaType.TextPlain) {
                "some plain text"
            } else {
                "some json"
            }
            Future.successful(Right(str))
        case None =>
            // Implement a better error message and optionally make it go in the error channel
            Future.failed(new RuntimeException("Unable to provide requested type"))
    }
}
```

🚀 **Now, this works in all cases** and moreover it's compliant with Tapir's own computation (like for the response `Content-Type`).

