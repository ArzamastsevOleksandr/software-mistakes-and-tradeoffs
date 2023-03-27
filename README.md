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
- 