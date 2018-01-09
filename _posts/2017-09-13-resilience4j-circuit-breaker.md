---
layout: post
title:  "Resilience4j - a lightweight, flexible circuit breaker"
author: Philippa Main
---

We knew that our application would break if the database was down. 
More precisely, we knew that our service end-point would time out when we were logging to the database some 
not-essential-but-useful information for each item in the request’s large batch.

We knew this would happen because it did when Auto Trader ran a Disaster Recovery test in our test environment and 
we needed a story to fix it soon.

As the data being logged was not essential, we needed to implement a circuit breaker to surround the database call 
and allow us to continue with essential work even when the circuit tripped on the first logging failure.

Martin Fowler has a [post that explains circuit breakers nicely](https://martinfowler.com/bliki/CircuitBreaker.html).

A foray into Google turned up a few possibilities, the most interesting being

* Netflix’s [Hystrix](https://github.com/Netflix/hystrix/wiki) is a library intended to help control the interactions 
  between distributed services by adding latency tolerance and fault tolerance logic. It is a big library that pulls in 
  quite a lot of dependencies and does much more than we needed. 
  Interestingly, it is integrated with Spring Boot using the [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/) 
  package, although that wasn't a consideration for our Dropwizard application.
* [Resilience4j](https://github.com/resilience4j/resilience4j) is a lightweight fault tolerant library inspired by 
  [Hystrix](https://github.com/Netflix/hystrix/wiki) but designed for Java 8 and functional programming. 
  At first glance, Resilience4j looked new but it is actually a new name for the more mature Javaslang-Circuitbreaker.
  It is built on top of Vavr (formally Javaslang), a functional language extension to Java 8.


Enough looking. Time was short. Resilience4j it was.

## The Circuit Breaker
The Resilience4j circuit breaker works in a beautifully simple and flexible way by decorating a Function, Supplier, 
Consumer, Runnable with a CircuitBreaker. You can then go on to decorate that with a whole load of other things,
which I will expand on below.

We used a custom circuit breaker because we wanted it to trip straight away:
```java
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
    .failureRateThreshold(1)
    .waitDurationInOpenState(Duration.ofMillis(120000))
    .ringBufferSizeInHalfOpenState(1)
    .ringBufferSizeInClosedState(1)
    .build();
CircuitBreaker circuitBreaker = CircuitBreaker.of("a-custom-circuit-breaker", circuitBreakerConfig);
```
but the default circuit breaker provided is even simpler to create:
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("a-default-circuit-breaker");
```

Then to use it, we just needed to decorate our DAO’s create() call with the circuit breaker. The original call
```java
dataLoggingDao.create(dataToLog);
```
turned into
```java
CheckedConsumer<AlgorithmResultSummary> recordCreator = dataLoggingDao::create;
CircuitBreaker.decorateCheckedConsumer(circuitBreaker, recordCreator)
              .accept(dataToLog);
```
… and that was it. A simple, clean circuit breaker around our problem piece of code.

## Looking more deeply into the circuit breaker

The default circuit breaker is equivalent to this custom circuit breaker:
```java
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .ringBufferSizeInHalfOpenState(10)
    .ringBufferSizeInClosedState(100)
    .build();
```
The ring buffers store the success / failure status of the most recent calls. There are two ring buffers. The 
closed-state ring buffer stores the status of calls made while the circuit is closed, i.e. the circuit breaker is not 
tripped. The half-open-state ring buffer stores the status of calls made while the circuit is in the half open state, 
i.e. after the circuit breaker has been tripped. The half-open-state ring buffer must be filled before the circuit 
can be considered to be closed again.

`failureRateThreshold` is the percentage of failures in the closed-state ring buffer above which the circuit breaker 
should trip open and start short-circuiting calls. The circuit breaker will not trip before the closed-state ring 
buffer has been filled.

`waitDurationInOpenState` is the amount of time the circuit breaker should stay open and short-circuit calls after it 
has been tripped. After this time, the circuit breaker moves to the half open state.

`ringBufferSizeInHalfOpenState` is the size of the half-open-state ring buffer.

`ringBufferSizeInClosedState` is the size of the closed-state ring buffer.

## The Circuit Breaker Registry

There is a registry supplied for you to manage your circuit breaker instances if you so wish. You create a 
`CircuitBreakerRegistry` with the circuit breaker configuration you wish to use as a default:

```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.of(circuitBreakerConfig);
```
then request circuit breakers from the registry as required. If you wish to have a circuit breaker with a different 
configuration you can specify one, otherwise you will get a circuit breaker that has the configuration that you used 
when creating the registry:

```java
// Get a CircuitBreaker from the CircuitBreakerRegistry with configuration that you when creating the registry
CircuitBreaker customCircuitBreaker1 = circuitBreakerRegistry.circuitBreaker("custom-circuit-breaker-1");

// Get another CircuitBreaker from the CircuitBreakerRegistry with configuration that you when creating the registry
CircuitBreaker customCircuitBreaker2 = circuitBreakerRegistry.circuitBreaker("custom-circuit-breaker-2");

// Get a CircuitBreaker from the CircuitBreakerRegistry using a different custom configuration
CircuitBreaker customCircuitBreaker3 = circuitBreakerRegistry.circuitBreaker("custom-circuit-breaker-3", otherCircuitBreakerConfig);
```

## More Than a Circuit Breaker

Because Resilience4j works by applying decorators to your consumers, functions, runnables and suppliers, you can 
combine the decorators in a very powerful way.

The core modules give you a circuit breaker, a rate limiter, a bulkhead for limiting the amount of parallel 
executions, an automatic retry (sync and async), response caching and timeout handling. There are add-on modules to 
give you metrics and more.

The resilience4j GitHub page at [https://github.com/resilience4j/resilience4j](https://github.com/resilience4j/resilience4j) 
gives some good examples of how the modules can be used:
* A circuit breaker with retry that handles any exceptions, and you can also configure a custom back-off algorithm
* A rate limiter
* A bulkhead
* A cache
* Adding metrics
* Consuming CircuitBreaker, RateLimiter, Cache and Retry events

## If You’re Interested
Find out more at:

GitHub page: [https://github.com/resilience4j/resilience4j](https://github.com/resilience4j/resilience4j)

Very good documentation page: [http://resilience4j.github.io/resilience4j](http://resilience4j.github.io/resilience4j)
