# software-mistakes-and-tradeoffs

These are my notes on
the [Software Mistakes and Tradeoffs](https://www.manning.com/books/software-mistakes-and-tradeoffs) book. The code
examples are written in Java.

## Chapter 1: Introduction

> Always validate your assumptions about the code performance, depending on whether it is executed in the single or
> multithreaded context.

Software design is all about tradeoffs. Choosing one direction almost always prevents you from moving in the other
direction.

### Testing strategies

Consider the following component:

```java
public class SystemComponent {
    public int publicApiMethod() {
        return privateApiMethod();
    }

    private int privateApiMethod() {
        return complexCalculations();
    }

    private int complexCalculations() {
        // complex logic
        return 0;
    }
}
```
Usually, the `publicApiMethod` test is enough, but if the `complexCalculations` has many possible execution paths we might want to test it by:
* changing the access modifier to `public` (the users might start using the `complexCalculations` method directly which is not intended)
* use the package-private modifier, that allows the tests to access this method and prevent users from doing it

Time is a limiting factor in software development, and it affects our decision on the unit/integration test proportions.

##### todo: 
create a table with pros/cons

**Unit tests**:
- are quick
- provide fast feedback 
- take less time to develop
- do not test the whole system and the moving parts
- validate a single aspect of a single component

**Integration tests**:
- are slower
- check the whole system and the interaction between its components
- take more time to develop
- validate multiple aspects of multiple components
- might require a complex infrastructure setup if N services are required to test the end-to-end flow

**Personal observations**: (add your own input on unit/integration tests)

### Code design patterns

Common design patterns are meant to improve the code quality. However, their usage depends on the context.

Consider the [Singleton](https://refactoring.guru/design-patterns/singleton) pattern.

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

The `Singleton` implementation:

- allows you to access the instance via a `getInstance` method and ensures that only a single instance is created
- does not work in a multithreaded environment

```java
public class SystemComponentSingletonSynchronized {
    private static SystemComponent instance;

    private SystemComponentSingletonSynchronized() {
    }

    public static synchronized SystemComponent getInstance() {
        if (instance == null) {
            instance = new SystemComponent();
        }
        return instance;
    }
}
```

The `SystemComponentSingletonSynchronized` implementation:

- works in a multithreaded environment
- leads to [thread contention](https://docs.oracle.com/javase/tutorial/essential/concurrency/sync.html) since
  the `getInstance` call acquires a monitor on the object which does not make much sense once the underlying instance
  has already been created
- the caller might store the `getInstance` result in a variable once and avoid calling the `getInstance` again to avoid
  thread contention 

By utilizing the `double-checked locking` technique we can get rid of the unnecessary performance degradation:

```java
public class ThreadSafeSingleton {
    private volatile static ThreadSafeSingleton instance;

    public static ThreadSafeSingleton getInstance() {
        if (instance == null) {
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) {
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
}
```

The `ThreadSafeSingleton` implementation:

- reduces the need for synchronization
- only has a performance degradation on startup when multiple threads try to initialize the `ThreadSafeSingleton`
  instance

Another approach is to use `thread confinement` which is not a singleton on the application level, but a singleton on
the thread level. There is no thread contention as the object is owned by one thread and not shared:

```java
public class ThreadLocalSystemComponent {
  private static final ThreadLocal<ThreadLocalSystemComponent> VALUE = ThreadLocal.withInitial(ThreadLocalSystemComponent::new);
}
```

The `ThreadLocalSystemComponent` implementation:

- no thread contention
- increased complexity
- might not be suitable to your needs

### Microservices vs Monolithic architectures

Microservices:

- allow for horizontal scaling to serve the traffic surge (given that the underlying data access layer can scale up and
  the efficient load balancing and service registry are used to distribute the traffic evenly)
- there may be an upper threshold of scalability after which adding new nodes does not give improvement in throughput (
  caused by scalability limit of the underlying components like database, queue, network, bandwidth etc) 
- ideally, separate teams can work independently on separate domains in the form of microservices
- ideally, no coordination at the codebase level between the teams
- ideally, more frequent less risky deployments (less features are deployed at once)
- many moving parts, increased complexity

Monoliths:

- allow for vertical scaling by adding more CPUs, memory, disk capacity (has a hard limit that can be quickly reached)
- the codebase can be shared by many teams
- high chance of code conflicts
- less frequent more risky deployments (more features are deployed at once)

## Chapter 2: Code duplication is not always bad: code duplication vs flexibility

> We can calculate the cost of coordination within teams using Amdahl's law.

The DRY principle suggests to avoid code duplication for the software to contain fewer bugs and be more flexible.
Overusing DRY may be dangerous. This principle is reasonable in monolithic architectures when the whole codebase resides
in a single repository. When used in microservices architectures the shared components might lead to the system coupling
and increase the amount of coordination between the teams.

If TeamA and TeamB are working on 2 separate services they can deliver functionality quickly and work independently with
no need for synchronization between the teams.

> [Amdahl's law](http://mng.bz/OG4R): the less synchronization is needed (there is a more parallel portion of work), the
> more gain we get from adding more resources.

The Amdahl's law can be adapted to team members working on a specific task. The synchronization reducing parallelism may
be the time spent on meetings, merge problems, etc.

When the code is duplicated, it is developed independently by both teams, and there is no synchronization needed between
those teams. Adding a new team member to a team would therefore increase performance. This situation differs when
reducing code duplication and the two teams need to work and block each other on the same piece of code.

When 2 teams are duplication the same components in different services:

- code/work duplication may lead to more bugs and mistakes
- no knowledge sharing between the teams, when TeamA fixes a bug it does not mean that TeamB does the same
- less coordination might lead to faster team progression

### Shared libraries

Once the common code is a part of the shared library the services will fetch it at a build time. Benefits:

- the library bug fixes are immediately available to all clients
- the teams can cooperate and contribute to the overall library quality
- no duplication of work
- calling library components is usually fast as the executed code resides in the same JVM

Disadvantages:

- a new deployment pipeline needs to be set up (package a JAR and publish it to the repository manager)
- someone needs to evolve the library and take ownership of its quality
- the cost of adding a first shared library might be high
- the new library must be advertised across the company for everyone to adapt it
- libraries should follow semantic versioning, backward compatibility between major versions is hard

> The shared library must have as few direct dependencies as possible since several libraries might transitively pull
> different 3rd party library versions when used in a single service and lead to conflicts.

### Code extraction to a separate microservice

If the functionality that is duplicated can be captured as a separate business domain, we can consider creating yet
another microservice that exposes the functionality as an HTTP API. If your application does not have high performance
requirements, making one additional HTTP call should not be problematic (assuming that this request is within a cluster
or a closed network and not to some random server on the internet that might be on the other side of the globe). Instead
of making a call to the library an API call to a separate service is made.

Disadvantages:

- the service needs to be deployed and will consume paid resources
- service maintenance has costs (monitoring, alerting - react accordingly)
- APIs should be versioned and backward compatible
- every API call is an HTTP round trip. If the API logic is straightforward it might be better to execute on the client
  side. If the API logic is heavy/complex - the API call price might be neglected.
- the more intermediate API calls are made, the higher resulting latency observed by the caller is
- need to account for cascading failures and service unavailability

### Low level code inheritance

Two code components might have the same or similar code, and we might feel the need to extract the common logic into the
abstract class. However, the fact that two things look identical does not mean they will solve the same business goal.
They may evolve differently. This is **incidental** duplication.

It’s usually easier to merge two concepts into one if they turn out to be the same rather than to separate them if they
turn out to be different.

Sometimes, what looks like duplication is just two different things that happen to be treated the same way in current
requirements but which may vary later and shouldn’t be treated as equivalent. It may be difficult to distinguish between
those two situations at the beginning of a system design. So consider postponing getting rid of duplication before the
bigger picture is clear.

Sometimes starting from an abstraction and adapting all possible usages to it may not be optimal. Instead, we can
implement our system by creating independent components and letting them live independently for some time (even if it
requires some code duplication). Later, we may begin to see some common patterns between those components, and
abstraction may emerge. That might be the proper time to remove duplication by creating some abstraction instead of
starting from it.

## Chapter 3: Handling errors

The code can fail in a variety of ways. The code should be **fault-tolerant** whenever possible. Tackling any possible
error explicitly in the code makes it unmaintainable, however.

Not every error should be recovered from. **Let it crash** philosophy states it is better not to recover from critical
failures. If an Out Of Memory error crashes the app - the supervisor simply restarts it. This allows not to bother with
the OOM error handling in the code.

### Exceptions

The Java Exception hierarchy is as follows:

`Object` => `Throwable` (class)

`Throwable` => `Error` (`OutOfMemoryError`, `StackOverflowError`) - a critical problem, often should not be handled
directly.

`Throwable` => `Exception` (signal problems in the code)

`Exception` => `Checked` (`IOException`, `InterruptedException`) - the compiler forces you to explicitly handle these.
Use these when there is a way to recover.

`Exception` => `Unchecked` (`IllegalArgumentException`, `NullPointerException`) - the compiler does not require explicit
handling. Often it is better to fail fast than to try and recover.

Some languages have unchecked exceptions only. Be careful to not propagate the thrown exception to the main application
thread as it might crash the application.

#### Catch all vs granular handling

Let's consider a `throwingMethod` that throws two checked exceptions:

```java
public class Throwing {
  public void throwingMethod() throws FileAlreadyExistsException, InterruptedException {
    // throwing logic
  }
}
```

Here are ways to handle this:

```java
public class NormalGranularity {
  public void demo() {
    try {
      new Throwing().throwingMethod();
    } catch (FileAlreadyExistsException e) {
      log.error("File already exists", e);
    } catch (InterruptedException e) {
      log.error("Interrupted", e);
    }
  }
}
```

Two `catch` blocks allow providing separate exception handling paths.

Benefits:

- code is readable and provides the necessary details
- each exception type is handled in a different way

We could catch all exceptions:

```java
public class CatchAll {
  public void demo() {
    try {
      new Throwing().throwingMethod();
    } catch (Exception e) {
      log.error("Problem ", e);
    }
  }
}
```

Benefits:

- if the intended behavior is to handle ANY exceptions and proceed further

Disadvantages:

- unintended exceptions can be caught
- no granular information about exception types is present at compile time

We could handle different exceptions in the same way but still explicitly describe them to the user:

```java
public class MultiCatch {
  public void demo() {
    try {
      new Throwing().throwingMethod();
    } catch (IOException | InterruptedException e) {
      log.error("Problem ", e);
    }
  }
}
```

#### Checked vs unchecked exceptions handling in a public API

Annotating method APIs with exceptions clearly states the reasons for method failures and allows to choose the handling
strategy at compile time. If an exception is not declared by the method - the caller might not know about it and the app
would fail at runtime unexpectedly. Declaring exceptions as a part of the method's API is verbose, however. Checked
exceptions force the method caller to use the try-catch blocks. It is possible, however, to wrap the method throwing a
checked exception into a wrapper throwing an unchecked exception.

Advantages of declaring checked exceptions as the part of the method API:

- explicit contract that allows the callers to reason about the method behavior without looking at implementation
- no surprise failures at runtime

When writing both called and caller code on your own it is reasonable to use unchecked exceptions as you will not have
surprises.

When creating a public API that is called by the code unknown to you - using the checked exceptions is reasonable to
clearly state API behavior.

#### Exception handling anti-patterns

##### Swallowing an exception

The swallowed exception never propagates up, risking silent failure of the system. You can log the error and proceed at
least.

```java
public class Swallow {
  void demo() {
    try {
      throwingMethod();
    } catch (Exception ignored) {
      // do nothing
    }
  }
}
```

##### Printing the stack trace to the standard output

The exception content should be logged or written to the file. If it is written to the standard output we risk to lose
the exception details.

#### Exceptions from third-party libraries

Avoid tight coupling with the third-party library and do not propagate the library-specific exceptions across the
codebase. Define a custom domain exception or wrap the throwing code into a custom wrapper that handles the thrown
exceptions. In the future, if the library change is not backward compatible, the codebase is minimally impacted.

#### Performance considerations

Exceptions are objects that contain a stack trace. Constructing and throwing/catching exceptions is a relatively slow
operation. Getting a stack trace is a considerably slower operation. Logging exceptions is a much slower operation since
the stack trace is transformed into a string.

If performance is critical consider avoiding examining the stack trace and especially logging it.

## Chapter 4: Balancing flexibility and complexity

> The higher flexibility your API has, the more code/execution model complexity you will introduce.

When designing a public API it is better to start small and add based on the end user's input rather than implementing
a lot of features upfront. Removing the features once they are exposed to the public is difficult.

When deciding which features to retain or remove
the [A/B testing](https://www.optimizely.com/optimization-glossary/ab-testing/) can be conducted.

When designing a library for the internal organization usage we need to oversee the required list of features. If not
much was predicted and the API is not extensible - we end up changing the library API often. If everything is accounted
for in advance - the library might be too complicated and over engineered.

Often the engineers develop the initial code with many design patterns enabling the component extensibility in the
future - this risks introducing many abstraction levels adding complexity to the system.

> A rule of thumb: start simple and improve gradually once the requirements get clearer.

When designing a library try to avoid the tight coupling with other third-party library components. Instead, expose the
generic interfaces and allow the clients to provide the implementations. This increases the library flexibility, but it
also increases the burden on the client side. A reasonable solution would be to provide a default implementation in the
library and allow the clients to override it. Abstracting everything, however, is not an option as some third-party
dependencies are too complex and have many moving parts.

If we try to foresee the exact use case, we might introduce fairly generic patterns, such as **listeners** or **hooks APIs**. 

When using a *hooks API*, you need to guard against unpredictable usage - your code should expect any exception. You need
to consider the thread execution model of your API extension points. If your design allows synchronous client calls,
assume that some clients will block these, impacting the SLA of your component and resource usage (threads). But
introducing asynchronous logic working in parallel in your code adds complexity. You need to maintain a dedicated thread
pool and monitor it. Even if you make your processing parallel, you cannot reduce the additional latency to zero. 

The *listener API* is similar to the hooks API, but it does not involve blocking between phases of your component's
execution. The signals you emit are asynchronous and should not impact your component's latency. You need to be careful
about the state emitted to a caller component that you do not own and assert that the state is immutable.

## Chapter 5: Premature optimization vs optimizing the hot path

For a lot of use cases **premature optimization is evil**.

> **SLA** helps manage customer expectations, and it specifies:
>
> * amount of traffic the service should handle
>
> * number of requests it needs to execute with a latency lower than a specific threshold

The **hot path** is the code that does most of the work and is executed for the majority of user requests.

By having enough data, rational decisions can be made allowing us to make non-negligible performance improvements
before the system goes live. The data should come from performance benchmarks executed before the application is
deployed to production. Expected traffic can be modeled when an SLA and the real production traffic expectations of the
system are defined. When we *have enough data* that backs up our experiments and hypotheses, *optimization is no longer
premature*.

### When premature optimization is evil

- expected traffic data is absent
- code is optimized based on false assumptions

Code optimization often leads to increased complexity. Sometimes performance needs to be traded over maintainability and
complexity.

### Optimizing hot paths

Example: there are `process-request (PR)` and `modify-schema (MS)` endpoints and the `N` times each endpoint is called
is based on the empirical data/SLA of the service. `PR` is called 10000/s with a latency of 200ms/request. `MS` is
called 10/s with a latency of 500ms/request. Which of these should we optimize first? Based on the req/s data we can
calculate the following:

* reducing the `PR` latency by 20ms (10%) gives us the overall latency reduction of `10000 * (200ms - 180ms) = 200000ms`
  .

* reducing the `MS` latency by 250ms (50%) gives us the overall latency reduction: `10 * (500ms - 250ms) = 2500ms`.

Conclusion: optimizing the `PR` endpoint leads to **80 times** more savings than optimizing the `MS` endpoint.

> Optimizing the **hot path** is crucial for the overall application performance.

Another way to decide which path to optimize is to calculate the total latency weight: the `MS` weight is `10 * 500 /
(10000 * 200 + 10 * 500) = 0.24%`, while the `PR` weight is `99.76%`.

**Conclusion**: optimize the `PR` first.

#### Pareto principle in the context of software

A small fraction of code delivers a substantial proportion of the value produced by the software (~80% of the value and
work that our system performs is delivered by only ~20% of the code).

If linear behavior is true, every path in the code has the same importance. Adding a new component to the system means
that the value delivered to the clients increases proportionally. In reality, every system has a core functionality that
provides the most value for the core business. The rest of the functionalities are not crucial and do not produce much
value (say, 20%), but they require 80% of the time and effort to build. The numbers are different for every system, but
the conclusion is:

> Optimizing a smaller part of the codebase might impact most of the clients.

When creating a new system try to define the SLA requirements with expected upper bounds of traffic that the system can
handle. Once those numbers are in place, create performance tests that simulate real-world traffic.

#### Configuring the number of concurrent users (threads) for a given SLA

Let's aim for the service that handles `10000 req/s` with an average latency of `50ms`. `1 thread` can
serve `1000 / 50 = 20` requests. The amount of threads required to handle such a load is `10000 / 20 = 500` given that a
single thread on average handles `20 req/sec`. `500` threads are required by the test tool to saturate the
system/network with traffic.

#### Final thoughts

Critical code paths can be measured for the number of invocations and the time it takes to execute the path. With those
numbers, we can detect the hot path and calculate performance gains from optimizing a small part of the code. Most of
the systems follow the Pareto principle, so optimizing the hot path, impacts and delivers improvements to the majority
of the clients.

#### A side note

A Linux command to get 100 random words from a file:

> sort -R words.txt | head -n 100 > result.txt

## Chapter 6: Simplicity vs cost of maintenance

> Encapsulating the downstream component settings from the clients allows evolving the APIs in a backward compatible
> way.

The configuration mechanism of our system is an entry point that exposed to the clients.

We can abstract away every downstream component (any element used by the system for which we are creating a UX) and
not expose any of those component settings directly with the tool that interacts with the user. This will improve the UX
of our system, but it will require substantial maintenance.

On the other side of the spectrum, we can directly expose all dependent settings. This option does not require much
maintenance from us, but such an approach has a couple of tradeoffs. First, we are tightly coupling our clients with the
downstream component used by our service or tool. This makes it hard to change the component. Additionally, handling
changes to the downstream component in a UX-friendly (and backward compatible) way will be difficult if not impossible.

todo: reiterate on this chapter

## Chapter 9: Libraries you use become your code

When an existing third-party library is chosen and used it in the codebase, we take full responsibility for this piece of
software that we did not develop.

Some libraries/frameworks, such as [Spring](https://spring.io/),
favor [convention over configuration](https://docs.spring.io/spring-framework/docs/3.0.0.M3/reference/html/ch16s10.html)
. This pattern allows to start using a library right away without any necessary configuration. It trades off explicit
settings for the UX simplicity. Be aware of this tradeoff and its limitations. The prototyping/experimentation is easier
when using software components that do not require a configuration upfront. 

> **Framework** and **library** concepts are often used interchangeably.
>
> A **framework** provides a *skeleton* for building the app, but the actual logic is implemented in the app itself.
>
> A **library** already implements *some* logic, and it is *only* called from the client code.

### Beware of the defaults

Let’s consider a scenario of using a third-party library responsible for HTTP calls, an [OkHttp](https://square.github.io/okhttp/), for example. 

```java
class Demo {
  void demo() throws IOException {
    Request request = new Request.Builder().url("http://localhost:9999").build();
    OkHttpClient client = new OkHttpClient.Builder().build();
    Call call = client.newCall(request);
    Response response = call.execute();
  }
}
```

The HTTP client is created as a *builder* with no explicit settings. The code is simple, but should not be used in
production like this. In the context of HTTP clients, *timeouts* are essential and *impact the performance and SLA* of
the app. If the service SLA is 100 ms, and the call to other services is made to fulfill the request, the other call
must complete faster than the service SLA. Choosing a proper timeout is important.

High timeouts are dangerous in the microservices architecture. One microservice may need to call multiple others. Others
may need to call the following microservices and so on. When one of the services hangs on the request processing, it may
cause *cascading failure* to other services. The higher the timeout is, the longer it may take to process a single
request, then there is a higher probability of cascading failure.

The code that needs to fulfill the request within *100 ms* will call this endpoint and might break the service SLA.
Instead of seeing a response in *100 ms* (success/failure), the clients are blocked for *10000 ms*. The thread that
executes these requests may be blocked for that long. One thread that is supposed to execute *~100 requests* (*10000 ms ÷
100 ms*) will be blocked and will not serve any other request for that amount of time, impacting the overall performance
of the service. This problem may not appear if only one thread waits too long. However, we will start noticing
performance issues if all or the majority of allocated threads are blocked for too long.

The default timeout settings cause this issue. The client should fail the request if it needs to wait for more than the
service SLA (*100 ms*). If the request fails, the client can retry it instead of waiting for *5000 ms* for any response.
The read timeout of the [OkHTTP](http://mng.bz/9KP7) is 10 seconds by default!

> Checking the defaults is **important**. 
> 
> In a real system, the timeouts must be configured according to the service SLA.


> When importing any third-party library be aware of its settings. 
> **Implicit** settings are OK for **prototyping**. **Explicit** settings are must-haves for **production systems**.

### Concurrency models and scalability

Let's imagine that the third-party library synchronous function is executed from a synchronous client code. The
execution flow is simple: `methodA` starts => `blocking function` is called => `methodA` resumes.

**What if `methodA` is asynchronous?** Every request is put into a queue. For example, when the web server needs to
process an HTTP request, the worker thread that accepts the request does not do the actual processing. It puts the data
into the queue which is then processed by a dedicated thread from the thread pool. When executing method calls from the
non-blocking code, be careful about calling the code you do not own since everything is executed within the same worker
thread. The async flow accepts the data and can do some preprocessing such as deserializing it from bytes. Next, it puts
the data information into a queue. This operation must be fast and nonblocking to not stall the processing of incoming
requests. We must know the execution of the code being called.

#### Using async and sync APIs

```java
// A blocking third-party API
interface EntityService {
  Entity load();

  void save(Entity entity);
}
```

It is hard to use this blocking API from within the non-blocking code you own. The app threading model may not allow any
blocking (Vert.x) as well. The easiest way is to *create a non-blocking wrapper around the blocking code*. The wrapper
may return promises that will be fulfilled in the future.

```java
class AsyncWrapper {
  final EntityService entityService;
  final ThreadPoolExecutor executor;

  AsyncWrapper(EntityService entityService) {
    this.entityService = entityService;
    executor = new ThreadPoolExecutor(1, 10, 100, TimeUnit.SECONDS, new LinkedBlockingDeque<>(100));
  }

  public CompletableFuture<Entity> load() {
    return CompletableFuture.supplyAsync(entityService::load, executor);
  }

  public CompletableFuture<Void> save(Entity entity) {
    return CompletableFuture.runAsync(() -> entityService.save(entity), executor);
  }
}
```

Let's analyze the code above: 
* async actions are executed in a separate thread
* a separate thread pool is used for thread execution
* the thread pool requires monitoring and fine-tuning (number of threads, queue)

If the library is blocking, its performance may be worse than if the library were async. **Wrapping the blocking code
may only postpone the scalability problem.**

> Async codebase is often more complicated than the synchronous one. If not used right, the async code might perform
> worse. For async code to work well all system parts often need to be async. The system is as strong as its weakest
> link.

```java
// Async third-party API
public interface EntityServiceAsync {
  CompletableFuture<Entity> load();

  CompletableFuture<Void> save(Entity entity);
}
```

All methods of this component are returning a promise meaning that the internals of the library are written in an async
way. We do not need to implement a sync => async translation layer. The thread pool used by the `EntityServiceAsync` is
encapsulated within the library. It may be already fine-tuned for the majority of use cases. **However, be aware of the
defaults.** The code is called from the client app. The underlying threads still occupy resources in the app. 

> It is easier to call async code from the blocking code than the other way around. 

> It is often more reasonable to pick the async version of the library over the blocking one. Even if your application
> flow is blocking today, you might convert it to async processing in the future for scalability/performance reasons. If
> you are already using an async library, it will be easier to migrate to the new flow. However, if you are using a
> blocking library, the migration will not be as simple. This will require a translation layer and thread pool
> management.
> Moreover, the code that was not written as an async call in the first place is often implemented differently.


#### Distributed scalability

> It is important to understand the third-party library scalability when using it in a distributed context.

Example: we need a library that provides scheduling capabilities. It needs to:
* run the task when a time threshold is met
* store the tasks in a persistence layer 
* update the task status upon its execution

Implementing this set of features for a single-node environment is straightforward. Integration
tests may validate the behavior of the scheduling library with the embedded database. 

The app using this library can be deployed to multiple nodes. The same task should not be processed by more than one
node. The state of jobs needs to be globally synchronized or partitioned to preserve system consistency. 

If the scheduling library is not implemented in a scalable way, all nodes will contend for the job record in the
database. The correctness of the changes could be achieved via using transitions or a global lock on a specific record
which may impact the performance. **This problem can be addressed if the scheduling library supports partitioning.** The
first node could be responsible for tasks in a given minute range, the other node for a different time period, and so
on. 

> When picking a library to be executed in a distributed environment, analyze it carefully to determine whether it will
> behave correctly.

Understanding the scalability model and knowing if it requires a global state allows to scale the app more easily. If
the library is not designed to work in a distributed environment, we are risking scalability and correctness problems.
Such problems often appear when the app is deployed to N nodes, where N is a higher than usual number of nodes. It often
happens when there is a surge in traffic related to a high business opportunity for the product. It often happens during
holidays. **This is not a time when we want to learn that the app relies on a library that does not scale.**

### Testability

> When picking a third-party library always have limited trust in it. Assume nothing.

The best form of validating a third-party library is via testing. We often cannot change the library code, however.  

When testing a component from our codebase, and the code does not allow us to impact some behavior, this is relatively
easy to change. If our code initializes an internal component without giving the caller the possibility to inject the
fake/mock value, the code can be refactored. When we use a third-party library, impacting the codebase may be hard or
not feasible. Even if we submit the change, the time from change to deployment can be substantial. 

> Before choosing a third-party library, validate its testability.

#### Testing library

Almost every third-party library has some internal state. If the library allows us to inject a different implementation,
that is a significant advantage.

Example: if the caching library is used it might evict the caches after some expiration period. Writing a test for
this behaviour would involve sleeping for specific amounts of time during the test execution. Test execution
becomes too slow. If the library provides the ability to inject the timers that handle the expiration periods - the
testability of such a library is greatly simplified since we can advance the internal timers easily without sleeping.

### Dependencies of third-party libraries

Every library or framework is written by engineers who may need to make a similar decision: **should we implement a
small part of the logic ourselves or use another library that provides that functionality?** When we import a library
that provides an HTTP client functionality, it should not rely on yet another library providing the same functionality.
The situation is different when engineers who create a library are not working on its core functionality. For example,
the HTTP client library may provide an out-of-the box JSON serialization/deserialization capabilities. Designers of the
HTTP client library may choose to use other third-party libraries to provide the JSON related functionality. It is a
reasonable decision, but it creates a couple of problems for the client app. Every class that is shipped with the HTTP
client (including their dependencies) will be visible to the client code, so we can use the JSON processing library via
transitive dependency. However, tight coupling is introduced between the app code and the library. The HTTP client
library may change the JSON processing library in the future, and the client code will break.

#### Semantic versioning and compatibility

Most libraries have adopted semantic versioning with a version string consisting of three parts: major, minor, and
patch. Any breaking change should be indicated by a change to the major part of the version string. The impact here is
that if the complete set of dependencies only uses the same major version for the JSON library, we should be able to
just use the latest of those versions everywhere. If there are multiple major versions involved, we need them to be
effectively independent dependencies.

#### Too many dependencies +

> Every library you import impacts your application.

Be aware that libraries use other libraries to provide non-core functionality. Examine the number of dependencies it
brings. There is a big difference between importing other libraries for hard-to-write, complex functionality and for
simple tasks that can be implemented easily from scratch. It is often not feasible for library creators to
perform shading for all dependencies, as it requires too much time and effort.

Every dependency imported into your app influences the target **fat jar** size. Seriously take into account the number
of dependencies your app has. The fewer the dependencies there are, the smaller the app will be, the faster it will
start, and also, it will be easier to deploy and manage. The runtime overhead will be lower because the built
application needs to be loaded in the machine's RAM that runs it.

> A **fat jar** is a self-contained application runnable with all required dependencies that can be easily run without
> external dependencies.

In the Java ecosystem, the [Maven-shade-plugin](https://maven.apache.org/plugins/maven-shade-plugin/) simplifies the
build process of the fat jar and provides a way to perform shading using the renaming technique. 

#### Reusing code +

If only a small part of the library functionality is required - consider copying that part to your code directly, add the tests for it, and take full ownership of the code. 
But now you need to be responsible for its bug fixes.

Forking the original library and develop needed functionalities is also an option but comes with maintenance problems as
you need to keep the fork up to date to include the bug fixes from the original codebase. Also, both code versions may
diverge, making it hard to keep them consistent.

#### Vendor lock-in +

Software gets deprecated. When using a library or service be aware there may be a need to migrate to a new solution in
the future. Try hiding the integration points behind an abstraction layer if you know that this probability is high.
When there is a need to switch the implementation, the change will not propagate to a lot of places in the code.
Instead, it will be encapsulated within the abstraction, and a code change will be needed only there.

When starting with a new library, observe how it impacts the app and architecture. The more invasive the integration is,
the harder it will be to change the vendor in the future. It is hard to hide every possible library and service
integration point behind abstraction in the real world. Some level of vendor lock-in is also hard to alleviate, but
strive to minimize it by picking libraries that do not require tight coupling with the app.

#### Library vs framework +

Often, we can abstract the library away easily. All calls to the HTTP service library can be hidden in the custom
wrapper. Later we can switch a library for another implementation without a considerable cost. We can also decide to
implement it ourselves and remove the dependency on this library.

Frameworks are invasive and require using their constructs throughout the app. The more framework imports are in the
codebase, the more tightly coupled they are. It is more difficult to change the framework to something else during the
lifecycle of the app compared to a library. 

#### Security and updates +

Ideally, we should perform security tests before deploying a new release of the app. Third-party dependencies evolve and
might contain new security vulnerabilities. If such is found, often a new version of the library is published with the
fix. We should upgrade the library ASAP. The longer we wait, the more time there is for the potential attacker to
exploit that vulnerability.

How can we find out if the third-party dependency had a security problem:
* check for [security vulnerabilities](https://www.cvedetails.com/)
* [automate security checks](https://dependabot.com/) that scan all third-party libraries and notify us when there is a problem

#### Decision checklist

Before using a third-party library consider the following:

| Parameter                                       | Question                                                                                                                                                               |
|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Configurability and defaults                    | Can we provide (and override) all critical settings?                                                                                                                   |
| Concurrency model, scalability, and performance | Does it provide an async API if our applications workflow is async as well?                                                                                            |
| Distributed context                             | Can it be safely run in a distributed context?                                                                                                                         |
| Investigating reliability                       | Are we choosing a framework or a library?                                                                                                                              |
| Testing                                         | How hard it is to test the code that uses this library? Does the library provide its own testing toolkit?                                                              |
| Dependencies                                    | What does the library depend on? Is it self-contained and isolated? Or does it download a lot of external dependencies, impacting the size and complexity of your app? |
| Versioning                                      | Does the library follow semantic versioning? Is it evolving in a backward-compatible way?                                                                              |
| Maintenance                                     | Is it popular and actively maintained?                                                                                                                                 |
| Integration                                     | How invasive is the integration with this library? How much do we risk being locked into a single vendor?                                                              |
| Security and updates                            | Is it frequently upgrading the downstream components to address their security vulnerabilities?                                                                        |
