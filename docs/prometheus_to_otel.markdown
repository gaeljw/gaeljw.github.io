---
layout: default
title: Migrating a JVM application from Prometheus metrics to OpenTelemetry
description: Concrete steps & tips to migrate your JVM applications from Prometheus metrics to OpenTelemetry
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

_Content of this article has been written with the OpenTelemetry version 1.21.x available as of December 2022. Post a comment if you notice something which is obsolete at the time of reading._

-----

# What is OpenTelemetry?

When someone mentions OpenTelemetry, it can actually refer to several things as OpenTelemetry is:

- a language-agnostic standard with a set of specifications for the 3 telemetry data types: metrics, traces and logs
- libraries for almost every language
- a protocol: OpenTelemetry Protocol (also known as OTLP)
- a “collector” to receive, process and export telemetry data

It’s also important to note that OpenTelemetry is vendor-agnostic and allows you to use any “backend” you want for storing/analyzing telemetry data: Prometheus, Datadog, New Relic…

## Why migrate?

The main reasons for migrating to OpenTelemetry are:

- be vendor-agnostic so that you don’t have to rewrite your applications if you change your backend provider in the future
- benefit from features not available in some vendors API
- unify the different types of telemetry data (metrics, traces and logs) in a consistent cross-language API
- standardize your telemetry data pipelines (thanks to the OpenTelemetry Collector)

## Architecture

Prometheus is pull-based, meaning it regularly scrapes your application to collect metrics.

Most of the time, your application exposes metrics via an HTTP endpoint. In some cases (mainly for batch workload and not recommended otherwise) your application use the Prometheus Pushgateway in push-mode as a proxy.

![Prometheus metrics scraping](./assets/prometheus_to_otel/PrometheusMetrics.drawio.png)

With OpenTelemetry, you have several architecture options depending on whether you want to use OpenTelemetry collector or not, and whether your application is built with OpenTelemetry API/SDK yet or not.

![OpenTelemetry x Prometheus](./assets/prometheus_to_otel/OTELProm.drawio.png)

- **Option A: your application uses the OpenTelemetry API/SDK but still exposes metrics as a Prometheus-compatible HTTP endpoint.**
- Option B: you add the OpenTelemetry collector as a proxy between your application and Prometheus. This makes sense as an intermediate step before moving to Option C.
- Option C: your application becomes totally agnostic of Prometheus and pushes metrics in the OTLP format to OpenTelemetry collector.

This article will focus on Option A which in my opinion is the easiest to start with OpenTelemetry: **nothing has to change in your architecture**, only the way metrics are implemented by your application will change.

Using OTLP and the OpenTelemetry Collector have benefits but this can be done as a second step and will require discussion and preparation with your Ops team (if you have one).

## Auto instrumentation vs. manual instrumentation

On the JVM, in the same manner as what is possible with Prometheus, OpenTelemetry can be used in two ways:

- auto instrumentation: add OpenTelemetry JAR as a Java agent and let it automatically exposes metrics for the JVM and the frameworks/libraries your application is using.
- manual instrumentation: if your application exposes custom metrics or you want more control

This article will focus on **manual instrumentation** as it’s the case where there is actually some work to do in your application.

Note that even if you use manual instrumentation, you can easily add metrics that would have been exposed by the auto instrumentation mode. We’ll cover that below.

## API vs. SDK

OpenTelemetry instrumentation is built around the concept of API and SDK:

- the API allows you to define metrics and how they are computed
- the SDK allows you to define how metrics are processed and exposed

If you are writing a library, you should only care about the API.

If you are writing an application, you should use both the API and the SDK. Nevertheless, all your “business code” should only rely on the API and not the SDK. We will see below how this is achieved in practice.

-----

# Migrating from Prometheus

## Dependencies

The first thing to do is to add the OpenTelemetry dependencies to your project as a replacement of the Prometheus ones.

For instance, if you are using Maven, you can remove the following dependencies:

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient</artifactId>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_hotspot</artifactId>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
</dependency>
```

And add the following ones as a replacement:

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-runtime-metrics</artifactId>
</dependency>
```

_Note: the exact list of dependencies (and versions) you are using might vary. Here I assumed Prometheus metrics were exposed through the bundled Prometheus HTTP server and you were exposing JVM metrics._

As always, check out the latest available versions of OpenTelemetry dependencies. As it’s still the early days, there are frequent updates and additions.

Each dependency declared above plays a different role:

- `opentelemetry-sdk` is the main dependency which will allow you to both define metrics and expose them. It’s a wrapper of other dependencies.
    
    It includes `opentelemetry-api`. If you are writing a library, you should depend only on the API.

    It also includes `opentelemetry-sdk-trace`, `opentelemetry-sdk-logs`, `opentelemetry-sdk-metrics`, and `opentelemetry-sdk-common`. You could technically only depend on the last two if you won’t work with traces but only with metrics.

- `opentelemetry-exporter-prometheus` is the dependency needed to expose metrics as a Prometheus Exporter. If we were to push metrics directly to OpenTelemetry Collector, this would not be needed.

- `opentelemetry-runtime-metrics` is the equivalent of Prometheus’ `simpleclient_hotspot`: it will allow exposing JVM metrics. If you don’t care about JVM metrics, you can skip this one.

## JVM metrics

JVM metrics give insights on CPU, memory, threads or garbage collector for instance.

With Prometheus, you would typically do the following to expose all available JVM metrics:

```java
import io.prometheus.client.hotspot.DefaultExports

DefaultExports.initialize();
```

The same can be achieved with OpenTelemetry with the following code:

```java
import io.opentelemetry.api.OpenTelemetry
import io.opentelemetry.instrumentation.runtimemetrics.*

OpenTelemetry otel = ... // Will be provided later (more below)

MemoryPools.registerObservers(otel);
BufferPools.registerObservers(otel);
Classes.registerObservers(otel);
Cpu.registerObservers(otel);
Threads.registerObservers(otel);
GarbageCollector.registerObservers(otel);
```

One important thing to note here is that you will need an `OpenTelemetry` instance to register the JVM observers. We’ll get back to how to obtain this instance and best practices later.

You can also notice we are talking about “**observers**” because these are metrics that will be “computed” only when asked for it. If you think of the memory instance, there is no metric that is continuously updated with the memory usage: it’s only when a value for the metric is asked (when Prometheus scrapes it for instance) that the value is actually computed or retrieved.

As of the day of writing these lines, there are slightly less JVM metrics exposed by OpenTelemetry than by Prometheus but all the major are most useful are. This might be of course evolve quickly if the community asks for it.

Lastly, OpenTelemetry also tries to standardize the metrics names across languages and frameworks. As a consequence, the JVM metrics names are different between Prometheus and OpenTelemetry. See below a non-exhaustive mapping:

```
Prometheus -> OpenTelemetry

# Labels
pool -> pool_name

# Metrics
jvm_buffer_pool_xxx -> process_runtime_jvm_buffer_xxx
jvm_classes_xxx -> process_runtime_jvm_classes_xxx
jvm_memory_pool_bytes_xxx -> process_runtime_jvm_memory_xxx
jvm_memory_bytes_xxx -> process_runtime_jvm_memory_xxx
jvm_threads_xxx -> process_runtime_jvm_threads_xxx
jvm_gc_collection_xxx -> process_runtime_jvm_gc_xxx
```

## Custom metrics definition

Custom metrics give insight about your application in a specific way. Typically metrics related to your business.

Let’s take the example of a simple counter. With Prometheus you would define and use it like this:

```java
import io.prometheus.client.*

// Definition
Counter myCounter = Counter
  .build()
  .name("myapp_mycounter")
  .help("Count of something")
  .labelNames("someLabel")
  .register();

// Usage
myCounter.labels("someLabelValue").inc();
```

With OpenTelemetry, the same can be achieved with following code:

```java
import io.opentelemetry.api.OpenTelemetry
import io.opentelemetry.api.common.{AttributeKey, Attributes}
import io.opentelemetry.api.metrics.LongCounter

OpenTelemetry otel = ... // Will be provided later (more below)

Meter meter = otel.getMeter("com.mycompany.myapp");

// Definition
LongCounter myCounter = meter
  .counterBuilder("myapp_mycounter")
  .setDescription("Count of something")
  .setUnit("something")
  .build();

// Usage
myCounter.add(1, Attributes.of("someLabel", "someLabelValue"));
```

As for JVM metrics previously, we need a instance of `OpenTelemetry` and we’ll see later (below) how to provide it.

The differences with Prometheus are:

- You need a `Meter` instance which kinda acts as a group of metrics. We won’t detail this part for now, you can just use your package name as meter name.

- The labels definition are not part of the counter itself but can be set “dynamically” when interacting with the counter

- The counter is typed, it can be a `LongCounter` or a `DoubleCounter` (you can use [`ofDoubles()`](https://javadoc.io/static/io.opentelemetry/opentelemetry-api/1.21.0/io/opentelemetry/api/metrics/LongCounterBuilder.html#ofDoubles()) to convert from the former to the latter).

Other than that, it’s pretty much the same.

Note that this is slightly more complex for histograms. We will cover them later below.

## Prometheus exporter

Once metrics are defined and used in your code, you want to expose them so that Prometheus can scrape them.

Here, we will consider that the metrics are exposed by a bundled Prometheus HTTP server.

With Prometheus API, this is only a matter of starting the HTTP server on a given port:

```java
import io.prometheus.client.exporter.HTTPServer

int prometheusHttpPort = ... // Any port you'd like
HTTPServer prometheusServer = new HTTPServer(prometheusHttpPort);

// Close the server when stopping your app
prometheusServer.close();
```

With OpenTelemetry, you would do pretty much the same:

```java
import io.opentelemetry.exporter.prometheus.PrometheusHttpServer

int prometheusHttpPort = ... // Any port you'd like
PrometheusHttpServer prometheusServer = PrometheusHttpServer.builder().setPort(prometheusHttpPort).build();

// Close the server when stopping your app
prometheusServer.close();
```

The only difference is that if you do nothing more, there’s nothing that “connects” all the metrics we defined previously and this Prometheus HTTP server.

We need to configure OpenTelemetry (SDK) to tie all this together.

## Tying it all together

So far we’ve seen how to define and use metrics as well as how to create a Prometheus exporter but with OpenTelemetry we need something to glue all this together.

This part only makes sense if you are building an application and using the OpenTelemetry SDK. If you’re building a library, this is not your concern.

Here is the code that we need to add:

```java
import io.opentelemetry.api.OpenTelemetry
import io.opentelemetry.api.common.Attributes
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.metrics.*
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes

// Create a Resource to identify your application
Resource myAppResource = Resource
  .getDefault()
  .merge(Resource.create(Attributes.of(ResourceAttributes.SERVICE_NAME, "my-app")));

// Create a MeterProvider
SdkMeterProvider meterProvider = SdkMeterProvider
  .builder()
  .setResource(myAppResource)
  // prometheusServer instance we defined previously, should be provided somehow
  .registerMetricReader(prometheusServer)
  .build();

// Glue everything together
OpenTelemetrySdk openTelemetrySdk = OpenTelemetrySdk
    .builder()
    .setMeterProvider(meterProvider)
    .buildAndRegisterGlobal();
```

The key thing to note in this code is the binding between a “**metric reader**”, in our case the Prometheus HTTP Server Exporter, and the `MeterProvider`. The rest is just boilerplate at this stage.

If later you’d like to send metrics to OpenTelemetry Collector rather than exposing them as a Prometheus HTTP server, you would only need to replace the **metric reader**.

> _**Okay, great, but… we referred to an `OpenTelemetry` instance previously and we still haven’t defined it!**_

Actually, we just did! The `OpenTelemetrySdk` instance above is also an instance of `OpenTelemetry`.

Now the question is how you make it available to the code we’ve seen previously defining metrics? The answer to this question depends on the way you build your application.

For instance if you use Dependency Injection, you would make `OpenTelemetry` injectable where needed and the piece of code defining the `OpenTelemetrySdk` could be a `Provider<OpenTelemetry>` :

```java
class MyMetrics {

  @Inject
  private OpenTelemetry otel;

  Meter meter = otel.getMeter("com.mycompany.myapp");

  // ... code defining metrics

}

class OpenTelemetryService implements Provider<OpenTelemetry> {

  // ... code to setup OpenTelemetry

  OpenTelemetrySdk openTelemetrySdk = ...

  public OpenTelemetry get() = {
    return openTelemetrySdk;
  }

}
```

Note that there’s also a “global instance” of `OpenTelemetry` which is made available by the `GlobalOpenTelemetry` class. However it’s recommended to not use it as it might hide issues due to initialization order of the several pieces that rely on it, especially if you are using auto-instrumentation as well.

-----

# Tips & tricks

## Histograms

As of the day of writing these lines, histograms are slightly more complex to migrate.

Let’s see an example with Prometheus:

```java
Histogram myHistogram = Histogram
    .build()
    .name("myapp_myhistogram")
    .help("Histogram of something")
    .labelNames("someLabel")
    .exponentialBuckets(1, 10, 9)
    .register();

// Using the histogram
myHistogram.labels("someLabelValue").observe(100d);
```

With OpenTelemetry, the histogram buckets cannot be specified when creating the histogram itself. The buckets are specified at the SDK level when tying everything together.

```java
// Defining the histogram
LongHistogram myHistogram = meter
  .histogramBuilder("myapp_myhistogram")
  .setDescription("Histogram of something")
  .setUnit("something")
  .ofLongs()
  .build();

// Using the histogram
myHistogram.record(100, Attributes.of("someLabel", "someLabelValue"));

// Defining the buckets (when instantiating the MeterProvider, in the class glueing everything)
SdkMeterProvider meterProvider = SdkMeterProvider
  .builder()
  .setResource(myAppResource)
  .registerMetricReader(prometheusServer)
  // This line is necessary to create buckets
  .registerView(
    // Select which histogram(s) to match
    InstrumentSelector
      .builder()
      .setType(InstrumentType.HISTOGRAM)
      .setName("myapp_myhistogram")
      .build(),
    // Declare the buckets
    View
      .builder()
      .setAggregation(Aggregation.explicitBucketHistogram(Arrays.asList(1, 10, 100, 1000, 10_000, 100_000, 1_000_000, 10_000_000, 100_000_000)))
      .build()
  )
  .build();
```

You have to register a “**view**” and a selector to choose on which “instruments” (counter, gauges, histograms…) it will be applied. In the example above, the selector match our histogram by its type and its name. Other selectors exist, for matching with regex for instance.

The idea behind this design is that the person that creates the instrument (the histogram in our case) is not necessarily the one that operates it. Defining the best buckets can depend on the context, especially for libraries.

However, when the person creating the instrument and operating it is the same, this creates the amount of complexity we’ve seen above. For this reason, there is an ongoing discussion to review this design and allow defining buckets at the histogram definition. This would be achieved using what is likely to be called a “hint”: the creator of the instrument could define what are the buckets for majority of use case and still let the ability to the operator to override it with the system of “views”.

> _You can follow this discussion and the progress on the following GitHub issue: [https://github.com/open-telemetry/opentelemetry-specification/issues/2229](https://github.com/open-telemetry/opentelemetry-specification/issues/2229)_

## UpDownCounters (Gauges)

An interesting addition in OpenTelemetry API is an instrument called `UpDownCounter` which can replace one of the use case of Prometheus gauges: a counter that can be increased or decreased.

Gauges still exist in OpenTelemetry for other use cases.

## Adding libraries or frameworks metrics

A lot of libraries and frameworks can be instrumented in the same way that we added JVM metrics previously.

For instance:

- Frameworks/servers: Akka, Jetty, Spring, Vertx
- Database connections pools: Apache DBCP, c3p0, HikariCP
- Databases: Cassandra, Elasticsearch
- Libraries: Guava, Hibernate

These instrumentations are available with an additional dependency, like:

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-hikaricp-3.0</artifactId>
</dependency>
```

The complete list of instrumentations supported by OpenTelemetry is available directly on their GitHub: [https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation).

Other instrumentations might be offered by the community in other projects.

-----

# Grafana dashboards

Community dashboards making use of OpenTelemetry metrics are still rare. Do not hesitate to contribute it if you write one.

Here are two dashboards that I created:

- JVM overview: [https://grafana.com/grafana/dashboards/17582-jmx-overview-opentelemetry/](https://grafana.com/grafana/dashboards/17582-jmx-overview-opentelemetry/)
- Hikari CP: [https://grafana.com/grafana/dashboards/17653-hikaricp-connection-pools-opentelemetry/](https://grafana.com/grafana/dashboards/17653-hikaricp-connection-pools-opentelemetry/)
