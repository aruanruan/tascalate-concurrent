[![Maven Central](https://img.shields.io/maven-central/v/net.tascalate.concurrent/net.tascalate.concurrent.lib.svg)](https://search.maven.org/artifact/net.tascalate.concurrent/net.tascalate.concurrent.lib/0.6.7/jar) [![GitHub release](https://img.shields.io/github/release/vsilaev/tascalate-concurrent.svg)](https://github.com/vsilaev/tascalate-concurrent/releases/tag/0.6.7) [![license](https://img.shields.io/github/license/vsilaev/tascalate-concurrent.svg)](http://www.apache.org/licenses/LICENSE-2.0.txt)
# tascalate-concurrent
The library provides an implementation of the CompletionStage interface and related classes these are designed to support long-running blocking tasks (typically, I/O bound) - unlike Java 8 built-in implementation, [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), that is primarily supports computational tasks.
# Why a CompletableFuture is not enough?
There are several shortcomings associated with [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) implementation that complicate its usage for blocking tasks:
1.	`CompletableFuture.cancel()` [method](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#cancel-boolean-) does not interrupt underlying thread; it merely puts future to exceptionally completed state. So if you use any blocking calls inside functions passed to `thenApplyAsync` / `thenAcceptAsync` / etc these functions will run till the end and never will be interrupted. Please see [CompletableFuture can't be interrupted](http://www.nurkiewicz.com/2015/03/completablefuture-cant-be-interrupted.html) by Tomasz Nurkiewicz
2.	By default, all `*Async` composition methods use `ForkJoinPool.commonPool()` ([see here](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--)) unless explicit [Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) is specified. This thread pool shared between all [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)-s, all parallel streams and all applications deployed on the same JVM. This hard-coded, unconfigurable thread pool is completely outside of our control, hard to monitor and scale. Therefore you should always specify your own Executor.
3.	Additionally, built-in Java 8 concurrency classes provides pretty inconvenient API to combine several [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)-s. `CompletableFuture.allOf` / `CompletableFuture.anyOf` methods accept only [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) as arguments; you have no mechanism to combine arbitrary [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)-s without converting them to [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) first. Also, the return type of the aforementioned `CompletableFuture.allOf` is declared as `CompletableFuture<Void>` - hence you are unable to extract conveniently individual results of the each future supplied. `CompletableFuture.anyOf` is even worse in this regard; for more details please read on here: [CompletableFuture in Action](http://www.nurkiewicz.com/2013/05/java-8-completablefuture-in-action.html) (see Shortcomings) by Tomasz Nurkiewicz

# How to use?
Add Maven dependency
```xml
<dependency>
    <groupId>net.tascalate.concurrent</groupId>
    <artifactId>net.tascalate.concurrent.lib</artifactId>
    <version>0.6.7</version>
</dependency>
```

# What is inside?
## 1.	Promise interface
The interface may be best described by the formula: 
```java
Promise == CompletionStage + Future
```

I.e., it combines both blocking [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)’s API, including `cancel(boolean mayInterruptIfRunning)` [method](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html#cancel-boolean-), AND composition capabilities of [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)’s API. Importantly, all composition methods of [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) API (`thenAccept`, `thenCombine`, `whenComplete` etc.) are re-declared to return `Promise` as well.

## 2. CompletableTask 
This is why this project was ever started. `CompletableTask` is the implementation of the `Promise` API for long-running blocking tasks. There are 2 unit operations to create a `CompletableTask`:

a.	`CompletableTask.asyncOn(Executor executor)`

Returns a resolved no-value `Promise` that is “bound” to the specified executor. I.e. any function passed to composition methods of `Promise` (like `thenApplyAsync` / `thenAcceptAsync` / `whenCompleteAsync` etc.) will be executed using this executor unless executor is overridden via explicit composition method parameter. Moreover, any nested composition calls will use same executor, if it’s not redefined via explicit composition method parameter:
```java
CompletableTask
  .asyncOn(myExecutor)
  .thenApplyAsync(myValueGenerator)
  .thenAcceptAsync(myConsumer)
  .thenRunAsync(myAction);
```  
All of `myValueGenerator`, `myConsumer`, `myActtion` will be executed using `myExecutor`.

b.	`CompletableTask.complete(T value, Executor executor)`

Same as above, but the starting point is a resolved `Promise` with the specified value:
```java
CompletableTask
   .complete("Hello!", myExecutor)
   .thenApplyAsync(myMapper)
   .thenApplyAsync(myTransformer)   
   .thenAcceptAsync(myConsumer)
   .thenRunAsync(myAction);
```  
All of `myMapper`, `myTransformer`, `myConsumer`, `myActtion` will be executed using `myExecutor`.

Additionally, you may submit [Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html) / [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) to the [Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) right away, in a similar way as with [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html):

```java
Promise<SomeValue> p1 = CompletableTask.supplyAsync(() -> {
  return blockingCalculationOfSomeValue();
}, myExecutor);

Promise<Void> p2 = CompletableTask.runAsync(this::someIoBoundMethod, myExecutor);
```

Most importantly, all composed promises support true cancellation (incl. interrupting thread) for the functions supplied as arguments:
```java
Promise<?> p1 =
CompletableTask
  .asyncOn(myExecutor)
  .thenApplyAsync(myValueGenerator)
  .thenAcceptAsync(myConsumer);
  
Promise<?> p2 = p1.thenRunAsync(myAction);
...
p1.cancel(true);
```  
In the example above `myConsumer` will be interrupted if already in progress. Both `p1` and `p2` will be resolved faulty: `p1` with a [CancellationException](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CancellationException.html) and `p2` with a [CompletionException](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionException.html).

## 3. DependentPromise
As it mentioned above, once you cancel `Promise`, all `Promise`-s that depends on this promise are completed with [CompletionException](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionException.html) wrapping [CancellationException](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CancellationException.html). This is a standard behavior, and [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) works just like this.

However, when you cancel derived `Promise`, the original `Promise` is not cancelled: 
```java
Promise<?> original = CompletableTask.supplyAsync(() -> someIoBoundMethod(), myExecutor);
Promise<?> derived = original.thenRunAsync(() -> someMethod() );
...
derived.cancel(true);
```
So if you cancel `derived` above it's [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) method, wrapping `someMethod`, is interrupted. However the `original` promise is not cancelled and `someIoBoundMethod` keeps running. This is not always a desired behavior, consider the following method:

```java
public Promise<DataStructure> loadData(String url) {
   return CompletableTask.supplyAsync( () -> loadXml(url) ).thenApplyAsync( xml -> parseXml(xml) ); 
}

...
Promise<DataStructure> p = loadData("http://someserver.com/rest/ds");
...
if (someCondition()) {
  // Only second promise is canceled, parseXml.
  p.cancel(true);
}
```

Clients of this method see only derrived promise, and once they decide to cancel it, it is expected that any of `loadXml` and `parseXml` will be interrupted if not completed yet. To address this issue the library provides [DependentPromise](https://github.com/vsilaev/tascalate-concurrent/blob/master/src/main/java/net/tascalate/concurrent/DependentPromise.java) class:
```java
public Promise<DataStructure> loadData(String url) {
   return DependentPromise
          .from(CompletableTask.supplyAsync( () -> loadXml(url) ))
          .thenApplyAsync( xml -> parseXml(xml), true ); 
}

...
Promise<DataStructure> p = loadData("http://someserver.com/rest/ds");
...
if (someCondition()) {
  // Now the whole chain is canceled.
  p.cancel(true);
}
```
[DependentPromise](https://github.com/vsilaev/tascalate-concurrent/blob/master/src/main/java/net/tascalate/concurrent/DependentPromise.java) overloads methods like `thenApply` / `thenRun` / `thenAccept` / `thenCombine` etc with additional argument:
- if method accepts no other [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html), like `thenApply` / `thenRun` / `thenAccept` etc, then it's a boolean flag `enlistOrigin` to specify whether or not the original `Promise` should be enlisted for the cancellation.
- if method accepts other [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html), like `thenCombine` / `applyToEither` / `thenAcceptBoth` etc, then it's a set of [PromiseOrigin](https://github.com/vsilaev/tascalate-concurrent/blob/master/src/main/java/net/tascalate/concurrent/PromiseOrigin.java) enum values, that specifies whether or not the original `Promise` and/or a `CompletionStage` supplied as argument should be enlisted for the cancellation along with the resulting promise, for example:

```java
public Promise<DataStructure> loadData(String url) {
   return DependentPromise
          .from(CompletableTask.supplyAsync( () -> loadXml(url + "/source1") ))
          .thenCombine( 
              CompletableTask.supplyAsync( () -> loadXml(url + "/source2") ), 
              (xml1, xml2) -> Arrays.asList(xml1, xml2),
              PromiseOrigin.ALL
          )          .
          .thenApplyAsync( xmls -> parseXmlsList(xmls), true ); 
}
```

Please note, then in release 0.5.4 there is a new default method `dependent` in interface [Promise](https://github.com/vsilaev/tascalate-concurrent/blob/master/src/main/java/net/tascalate/concurrent/Promise.java) that serves the same purpose and allows to write chained calls:

```java
public Promise<DataStructure> loadData(String url) {
   return CompletableTask
          .supplyAsync( () -> loadXml(url) )
          .dependent()
          .thenApplyAsync( xml -> parseXml(xml), true ); 
}
```

## 5. Timeouts
Any robust application requires certain level of functionality, that handle situations when things go wrong. An ability to cancel a hanged operation existed in the library from the day one, but, obviously, it is not enough. Cancellation per se defines "what" to do in face of the problem, but the responsibility "when" to do was left to an application code. Starting from release 0.5.4 the library fills the gap in this functionality with timeout-related stuff.

An application developer now have the following options to control execution time of the `Promise` (declared in Promise interface itself):
```java
<T> Promise<T> orTimeout(long timeout, TimeUnit unit[, boolean cancelOnTimeout = true])
<T> Promise<T> orTimeout(Duration duration[, boolean cancelOnTimeout = true])
```
These methods creates a new `Promise` that is either resolved sucessfully/exceptionally when original promise is resolved within a timeout given; or it is resolved exceptionally with a [TimeoutException](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/TimeoutException.html) when time expired. In any case, handling code is executed on the default asynchronous [Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html) of the original `Promise`.
```java
Executor myExecutor = ...; // Get an executor
Promise<String> callPromise = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor )
    .orTimeout( Duration.ofSeconds(3000) );
```
In the example above `callPromise` will be resolved within 3 seconds either successfully/exceptionally as a result of the `someLongRunningIoBoundMehtod` execution, or exceptionally with a `TimeoutException`.

The optional `cancelOnTimeout` parameter defines whether or not to cancel the original `Promise` when time is expired; it is implicitly true when skipped. So in example above `the someLongRunningIoBoundMehtod` will be interrupted if it takes more than 3 seconds to complete.

This is a desired behavior in most cases but not always. The library provides an option to set several non-cancelling timeouts like in example below:
```java
Executor myExecutor = ...; // Get an executor
Promise<String> resultPromise = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor );

// Show UI message to user to let him/her know that everything is under control
Promise<?> t1 = resultPromise
    .orTimeout( Duration.ofSeconds(2000) )
    .exceptionally( e -> {
      if (e instanceof TimeoutException) {
        UI.showMessage("Operation takes longer than expected, please wait...");
      }
      return null;
    }, false); 

// Show UI confirmation to user to let him/her cancel operation explicitly
Promise<?> t2 = resultPromise
    .orTimeout( Duration.ofSeconds(5000) )
    .exceptionally( e -> {
      if (e instanceof TimeoutException) {
        UI.showConfirmation("Service does not responding. Do you whant to cancel (Y/N)?");
      }
      return null;
    }, false); 

// Cancel in 10 seconds
resultPromise.orTimeout( Duration.ofSeconds(10), true );
```
Please note that the timeout is started from the call to the `orTimeout` method. Hence, if you have a chain of unresolved promises ending with the `orTimeout` call then the whole chain should be completed within time given:
```java
Executor myExecutor = ...; // Get an executor
Promise<String> parallelPromise = CompletableTask
    .supplyAsync( () -> someLongRunningDbCall(), executor );
Promise<List<String>> resultPromise = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor )
    .thenApplyAsync( v -> converterMethod() )
    .thenCombineAsync(parallelPromise, (u, v) -> Arrays.asList(u, v))
    .orTimeout( Duration.ofSeconds(5) );
```
In the example above `resultPromise` will be resolved successfully if and only if all of `someLongRunningIoBoundMehtod`, `converterMethod` and even `someLongRunningDbCall` are completed within 5 seconds. Moreover, in the example above only the call to `thenCombineAsync` will be cancelled on timeout, to cancel the whole tree please use the functionality of the `DependentPromise` class:
```java
Executor myExecutor = ...; // Get an executor
Promise<String> parallelPromise = CompletableTask
    .supplyAsync( () -> someLongRunningDbCall(), executor );
Promise<List<String>> resultPromise = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor )
    .dependent()
    // enlist promise of someLongRunningIoBoundMehtod for cancellation
    .thenApplyAsync( v -> converterMethod(), true )  
    // enlist result of thenApplyAsync and parallelPromise for cancellation
    .thenCombineAsync(parallelPromise, (u, v) -> Arrays.asList(u, v), PromiseOrigin.ALL)
    .orTimeout( Duration.ofSeconds(5) ); // now timeout will cancel the whole chain
```
Another useful timeout-related methods declared in Promise interface are:
```java
<T> Promise<T> onTimeout(T value, long timeout, TimeUnit unit[, boolean cancelOnTimeout = true])
<T> Promise<T> onTimeout(T value, Duration duration[, boolean cancelOnTimeout = true])
<T> Promise<T> onTimeout(Supplier<? extends T>, long timeout, TimeUnit unit[, boolean cancelOnTimeout = true])
<T> Promise<T> onTimeout(Supplier<? extends T>, Duration duration[, boolean cancelOnTimeout = true])
```
The `onTimeout` family of methods are similar in all regards to the `orTimeout` methods with the single obvious difference - instead of resolving resulting `Promise` exceptionally with the `TimeoutException` when time is expired, they are resolving it successfully with the `value` supplied (either directly or via [Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)):
```java
Executor myExecutor = ...; // Get an executor
Promise<String> callPromise = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor )
    .onTimeout( "Timed-out!", Duration.ofSeconds(3000) );
```
The example shows, that `callPromise` will be resolved within 3 seconds either successfully/exceptionally as a result of the `someLongRunningIoBoundMehtod` execution, or with a default value `"Timed-out!"` when time exceeded.

Finally, the `Promise` interface provides an option to insert delays into the call chain:
```java
<T> Promise<T> delay(long timeout, TimeUnit unit[, boolean delayOnError = true])
<T> Promise<T> delay(Duration duration[, boolean delayOnError = true])
```
The delay is started only after the original `Promise` is completed either successfully or exceptionally (unlike `orTimeout` / `onTimeout` methods where timeout is strated immediately). The resulting delay `Promise` is resolved after the timeout specified with the same result as the original `Promise`. Like with other timeout-related methods, delay is completed on the default asynchronous `Executor` of the original `Promise`. The latest methods' argument - `delayOnError` - specifies whether or not we should delay if original Promise is resolved exceptionally, by default this argument is true. If false, then delay `Promise` is completed immediately after the failed original `Promise`. 
```java
Executor myExecutor = ...; // Get an executor
Promise<String> callPromise1 = CompletableTask
    .supplyAsync( () -> someLongRunningIoBoundMehtod(), executor )
    .delay( Duration.ofSeconds(1000) ) // Give a second for CPU to calm down :)
    .thenApply(v -> convertValue(v));
    
Promise<String> callPromise2 = CompletableTask
    .supplyAsync( () -> aletrnativeLongRunningIoBoundMehtod(), executor )
    .delay( Duration.ofSeconds(1000), false ) // Give a second for CPU to calm down ONLY on success :)
    .thenApply(v -> convertValue(v));
```
You may notice, that delay may be introduced only in the middle of the chain, but what to do if you'd like to back-off the whole chain execution? Just start with a resolved promise!
```java
// Option 1
// Interruptible tasks chain on the executor supplied
CompletableTask.asyncOn(executor)
    .delay( Duration.ofSeconds(5) )
    .thenApplyAsync(ignore -> produceValue());

// Option2
// Computational tasks on ForkJoinPool.commonPool()
Promises.from(CompletableFuture.completedFuture(""))
    .delay( Duration.ofSeconds(5) )
    .thenApplyAsync(ignore -> produceValue());
```
As long as back-off execution is not a very rare case, the library provides the following convinient methods in the `CompletableTask` class:
```java
static Promise<Duration> delay(long timeout, TimeUnit unit, Executor executor);
static Promise<Duration> delay(Duration duration, Executor executor);
```

## 5. Polling and asynchronous retry functionality
Provided by utility class Promises but stands on its own
```java
static Promise<Void> poll(Runnable codeBlock, 
                          Executor executor, RetryPolicy retryPolicy)
static <T> Promise<T> poll(Callable<? extends T> codeBlock, 
                           Executor executor, RetryPolicy retryPolicy)
static <T> Promise<T> pollOptional(Callable<Optional<? extends T>> codeBlock, 
                                   Executor executor, RetryPolicy retryPolicy)
```
TBD

## 6. Utility class Promises
The class
provides convenient methods to combine several `CompletionStage`-s:

```java
static <T> Promise<List<T>> all([boolean cancelRemaining=true,] CompletionStage<? extends T>... promises)
````

Returns a promise that is completed normally when all `CompletionStage`-s passed as parameters are completed normally; if any promise completed exceptionally, then resulting promise is completed exceptionally as well

```java
static <T> Promise<T> any([boolean cancelRemaining=true,] CompletionStage<? extends T>... promises)
```

Returns a promise that is completed normally when any `CompletionStage` passed as parameters is completed normally (race is possible); if all promises completed exceptionally, then resulting promise is completed exceptionally as well

```java
static <T> Promise<T> anyStrict([boolean cancelRemaining=true,] CompletionStage<? extends T>... promises)
```

Returns a promise that is completed normally when any `CompletionStage` passed as parameters is completed normally (race is possible); if any promise completed exceptionally before first result is available, then resulting promise is completed exceptionally as well (unlike non-Strict variant, where exceptions are ignored if result is available at all)

```java
static <T> Promise<List<T>> atLeast(int minResultsCount, [boolean cancelRemaining=true,] 
                                    CompletionStage<? extends T>... promises)
```

Generalization of the `any` method. Returns a promise that is completed normally when at least `minResultCount`
of `CompletionStage`-s passed as parameters are completed normally (race is possible); if less than `minResultCount` of promises completed normally, then resulting promise is completed exceptionally

```java
static <T> Promise<List<T>> atLeastStrict(int minResultsCount, [boolean cancelRemaining=true,] 
                                          CompletionStage<? extends T>... promises)
```

Generalization of the `anyStrict` method. Returns a promise that is completed normally when at least `minResultCount` of `CompletionStage`-s passed as parameters are completed normally (race is possible); if any promise completed exceptionally before `minResultCount` of results are available, then resulting promise is completed exceptionally as well (unlike non-Strict variant, where exceptions are ignored if `minResultsCount` of results are available at all)

Please note that since library's version 0.5.3 it is possible to define explicitly whether or not to eagerly cancel remaining `promises` once the result of the invocation is known. The option is controlled by the parameter `cancelRemaining`. When ommitted, it means implicitly `cancelRemaining = true`.

The version 0.5.4 of the library adds additional overloads to aforementioned methods - now it's possible to supply a [list](https://docs.oracle.com/javase/8/docs/api/java/util/List.html) of `Promise`-s instead of the latest varagr parameter.

Additionally, it's possible to convert to `Promise` API ready value:

```java
static <T> Promise<T> success(T value)
```

...exception:

```java
static <T> Promise<T> failure(Throwable exception)
```

...or arbitrary `CompletionStage` implementation:

```java
static <T> Promise<T> from(CompletionStage<T> stage)
```

## 7. Extensions to ExecutorService API

It’s not mandatory to use any specific subclasses of `Executor` with `CompletableTask` – you may use any implementation. However, someone may find beneficial to have a `Promise`-aware `ExecutorService` API. Below is a list of related classes/interfaces:

a. Interface `TaskExecutorService`

Specialization of [ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html) that uses `Promise` as a result of `submit(...)` methods.

b. Class `ThreadPoolTaskExecutor`
A subclass of the standard [ThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html) that implements `TaskExecutorService` interface.

c. Class `TaskExecutors`

A drop-in replacement for [Executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html) utility class that returns various useful implementations of `TaskExecutorService` instead of the standard [ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html).

# Acknowledgements

Internal implementation details are greatly inspired [by the work](https://github.com/lukas-krecan/completion-stage) done by [Lukáš Křečan](https://github.com/lukas-krecan). The part of the polling / asynchronous retry functionality is adopted from the [async-retry](https://github.com/nurkiewicz/async-retry) library by [Tomasz Nurkiewicz](http://nurkiewicz.com/)

