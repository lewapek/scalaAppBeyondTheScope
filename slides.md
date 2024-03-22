
## What triggered this talk?

--

## Imperfection


--

## Scala gives us powerful tools

- ADTs
- `Option`s, `Either`s
- effects:
  - `ZIO`, cats `IO`, monix `Task`
- composability
- typeclasses, newtypes, higher-kinded types
- ...

---

## Final solution

- final?
  - constant improvement
  - bugfixes
- limited amount of time

--

## Current solution

- may never be final
- always imperfect
- even assuming your software is perfect
  - outer world is not

---

## Element of larger organism

- distributed system
- backing services
  - kafka, postgres, mongo, redis, ...
- information exchange
- app needs to run "somewhere"
  - target platform
    - vm?
    - kubernetes?

---

<!-- .slide: data-background-image="assets/world.avif" -->

---

## Cloud

```
          .-~~~-.
  .- ~ ~-(       )_ _
 /                     ~ -.
|                           \
 \                         .'
   ~- . _____________ . -~
                                .-~~~-.
                        .- ~ ~-(       )_ _
                       /                     ~ -.
                      |                           \
                       \                         .'
                         ~- . _____________ . -~
```

--

## Cloud

Different rules:
- apps are ephemeral
  - fast startup
  - graceful shutdown
  - frequent restarts
- stateless preferred
  - easier to scale horizontally
- containers everywhere
  - "self-containing" apps

---

## Fast startup

- started?
- fully initialized?
- ready to handle incoming requests?

--

## Fast startup

```text
Started          Initialized
|----------------|------------------------------------------>
                                                         time

  not ready yet      can handle incoming traffic
```

--

## Fast startup

- parallel init
  - faster
  - higher cpu usage at startup
- example: `ZLayer`

--

## Fast startup

```scala
  ZLayer.make[Requirements](
    CommonConfig.layer,
    prometheus.publisherLayer,
    prometheus.prometheusLayer,
    Bootstrap.sttpBackendLayer,
    ...,
    ProductService.layer,
    PostgresDatabase.transactorLive,
    Tracing.live,
    Baggage.live(),
    TracingService.layer,
    ContextStorage.fiberRef,
    JaegerTracer.live
  )
```

---

## Graceful shutdown

- usually by receiving SIGTERM
- refuse new requests
- finish current ones


--

## Release resources

- cats `Resource`
- ZIO `Scope` (`ZManaged`)
  - `ZIO.acquireReleaseExitWith(acquire)(release)(use)`
- effect systems work quite well here
  - in terms of signal handling and interruptions

--

## Graceful shutdown

```text
Shutdown signal      Graceful shutdown?       Forced shutdown 
|--------------------------------------------->
0s                                            30s 
```

--

## Graceful shutdown

- not implemented?
- force quit
  - data losses
  - resources not properly released
  - takes more time to close the app
    - deployment rollout

---

## How do we know if app works fine?

--

## How do we know if sth is wrong?

---

## Crashed?

- problem is visible
- not ideal but at least we know
- crashed app can be restarted
  - when?

--

## Exit code

```scala
  override val run =
    program.catchAll { error =>
      ZIO.logError(s"Ooops, an error occurred: ${error.show}")
    }
```

--

## Exit code

```scala
  override val run =
    program.catchAll { error =>
      ZIO.logError(s"Ooops, an error occurred: ${error.show}") *>
        exit(ExitCode(2))
    }
```

--

## Exit code

```scala
  override val run =
    program.tapError { error =>
      ZIO.logError(s"Ooops, an error occurred: ${error.show}")
    }
```

--

## Exit code

- returning 0 (success)
  - may prevent automatic restart

---

## Not crashed?

- still works
  - works fine?

--

## "hidden" failure?

detect it
restart / give time to recover

--

## but hidden failures happen sometimes

- don't cause whole app to crash
- they make the app unhealthy

---

## crashed > hidden failures?

---

<!-- .slide: data-background-image="./assets/this-is-fine.jpg" -->

---

## Healthchecks

- are you health?
- are you busy?

--

## Healthchecks

- executed periodically
- kubernetes example:
  - liveness probe - restart
  - readiness probe - mark not ready

--

## Healthchecks

```scala
  val postgres =
    Healthcheck(
      "postgres",
      for
        check <- fr"SELECT 1"
          .query[Int]
          .unique
          .transact(transactor)
          .timeout(postgresTimeout)
          .orDie
      yield check match
        case Some(1) => Healthcheck.Status.Ok
        case _       => Healthcheck.Status.Error
    )
  end postgres
```

--

## Healthchecks

- what can go wrong?
- too frequent restarts
- restart is (probably) not a permanent fix
  - interrupts other parts of the program
  - example: kafka consumers

--

## Healthchecks

- no restart (no problem) > crash / restart
- fine-grained control
  - local retries
  - restart part of your program

--

## Local retries

```scala
private val schedule =
  Schedule.recurWhileZIO[Any, Error] {
    case RateLimit => ZIO.logInfo("Rate limit exceeded").as(true)
    case _         => ZIO.succeed(false)
  } && {
    (
      Schedule.spaced(10.seconds).jittered && Schedule.recurs(7)
    ).orElse(
      Schedule.spaced(10.minutes).jittered && Schedule.recurs(7)
    )
  }
```

--

## Local retries

```scala
  ...
  someExternalCall
    .retry(schedule)
  ...
```

---

## Observability

---

## Monitoring, logs

- metrics / logs
- pull vs push based
- warning:
  - logs can singnificantly slow down your app

--

## Monitoring

```scala
val asyncJobsInProgress = Metric.gauge("jobs_running")
...
for
  _ <- ZIO.logInfo("Starting job")
  _ <- PrometheusMetrics.asyncJobsInProgress.increment
  ...
yield ...  
```

--

## Logs

```scala
ZIO.logAnnotate(
  LogAnnotation("key", "val"),
  LogAnnotation("key2", "val2")
) {
  ZIO.logInfo("Starting ...") *> effect
}
```

--

# Metrics

- monitoring / log based
- why?

--

## Metrics

- dashboards
- alerting

--

## Metrics

![Grafana example](/assets/grafana-screenshot.png "Grafana example")

--

## Metrics

- what else?

---

## (Auto)scaling

- stateless?
- exposes metrics / logs info

--

## (Auto)scaling

- cpu
- memory
- custom metric
  - app needs to expose
  - example: kafka lag
- benefits?
  - you can even save money!
  - auto scale down

---

## Tracing

- **traces** show the flow of the request
  - what happens next?
  - how much time it takes?
  - metadata? no problem

--

## Tracing

![Jaeger example](/assets/jaeger-example.png "Jaeger example")

---

## OpenTelemetry

- standrard
  - still fresh and evolving
  - not specific implementation
- *collection of APIs, SDKs and tools*

--

## OpenTelemetry

- traces
- metrics
- logs

--

## Tracing

- it's backend responsibility to support the standard
- backends are easier to replace
  - jaeger ...
- app needs to send traces to collector
  - libraries for popular laguages
  - automatic instrumentation available for some of them

--

## Tracing

```scala
...
    effect @@ tracing.aspects.extractSpan(
      TraceContextPropagator.default,
      carriers.input,
      spanName,
      SpanKind.SERVER
    ) @@ PrometheusMetrics.requestHandlerTimer
      .tagged("endpoint", spanName)
      .trackDuration
```

---

## Conclusions

- internal impl ==> outer view
- developer needs to help outer tools
  - to make use of their features
  - example
    - healthchecks
    - monitoring / tracing / logs
    - fast startup / graceful shutdown
    - exit code

