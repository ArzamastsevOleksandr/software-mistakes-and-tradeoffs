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


