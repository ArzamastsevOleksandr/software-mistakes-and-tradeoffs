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
