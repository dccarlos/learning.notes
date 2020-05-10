# 2. Thread safety

Surprisingly, concurrent programming isn’t so much about threads or locks, these are just mechanisms—means to an end.

 > Writing thread-safe code is, at its core, about managing access to _**state**_, and in particular to shared, _**mutable state**_.
 
Informally, an object’s _**state**_ is its data, stored in state variables such as instance or static fields. An object’s _**state**_ encompasses any data that can affect its externally visible behavior.

By _**shared**_, we mean that a variable could be accessed by multiple threads; by _**mutable**_, we mean that its value could change during its lifetime. What we are really trying to do is protect data from uncontrolled concurrent access.

_**Whether an object needs to be thread-safe depends on whether it will be accessed from multiple threads**_. This is a property of how the object is used in a program, not what it does. _**Making an object thread-safe requires using synchronization to coordinate access to its mutable state**_; failing to do so could result in data corruption and other undesirable consequences.

> The primary mechanism for synchronization in Java is the synchronized keyword, which provides exclusive locking, but the term “synchronization” also includes the use of volatile variables, explicit locks, and atomic variables.

If multiple threads access the same mutable state variable without appro- priate synchronization, _**your program is broken**_. There are three ways to fix it:

- Don’t share the state variable across threads.
- Make the state variable immutable.
- Use synchronization whenever accessing the state variable.

> It is far easier to design a class to be thread-safe than to retrofit it for thread safety later. When designing thread-safe classes, good object-oriented techniques—encapsulation, immutability, and clear specification of invariants—are your best friends.

It is always a good practice first to make your code right, and then make it fast. Even then, pursue optimization only if your performance measurements and requirements tell you that you must, and if those same measurements tell you that your optimizations actually made a difference under realistic conditions.

_**Is a thread-safe program one that is constructed entirely of thread-safe classes?**_ Not necessarily—a program that consists entirely of thread-safe classes may not be thread-safe, and a thread-safe program may contain classes that are not thread-safe. In any case, the concept of a thread-safe class makes sense only if the class encapsulates its own state. _**Thread safety may be a term that is applied to code, but it is about state, and it can only be applied to the entire body of code that encapsulates its state, which may be an object or an entire program.**_

## What is thread safety?
### Formal definition
> _A class is thread-safe if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization or other coordination on the part of the calling code._

If an object is correctly implemented, no sequence of operations—calls to public methods and reads or writes of public fields—should be able to violate any of its invariants or postconditions. No set of operations performed sequentially or con- currently on instances of a thread-safe class can cause an instance to be in an invalid state.

> Thread-safe classes encapsulate any needed synchronization so that clients need not provide their own.

### Example: A stateless servlet
```{java}
@ThreadSafe
public class StatelessFactorizer implements Servlet {
  public void service(ServletRequest req, ServletResponse resp) { 
    BigInteger i =   extractFromRequest(req);
    BigInteger[] factors = factor(i);
    encodeIntoResponse(resp, factors);
  } 
}
```

The transient state for a particular computation exists solely in local variables that are stored on the thread’s stack and are accessible only to the executing thread. One thread accessing a `StatelessFactorizer` cannot influence the result of another thread accessing the same `StatelessFactorizer`; because the two threads do not share state, it is as if they were accessing different instances. Since the actions of a thread accessing a stateless object cannot affect the correctness of operations in other threads, stateless objects are thread-safe.

> Stateless objects are always thread-safe.

## Atomicity
Atomic means something that executes as a single, indivisible operation.

What happens when we add one element of state to what was a stateless object? Suppose we want to add a “hit counter” that measures the number of requests processed. The obvious approach is to add a long field to the servlet and increment it on each request:

```{java}
@NotThreadSafe
public class UnsafeCountingFactorizer implements Servlet {
  private long count = 0;
  
  public long getCount() { return count; }
  
  public void service(ServletRequest req, ServletResponse resp) { 
    BigInteger i =   extractFromRequest(req);
    BigInteger[] factors = factor(i);
    encodeIntoResponse(resp, factors);
  }
} 
```

While the increment operation, ++count, may look like a single action because of its compact syntax, it is not atomic, which means that it does not execute as a single, indivisible operation. Instead, it is a shorthand for a sequence of three discrete operations: fetch the current value, add one to it, and write the new value back. This is an example of a _read-modify-write_ operation, in which the resulting state is derived from the previous state.

The possibility of incorrect results in the presence of unlucky timing is so important in concurrent programming that it has a name: a _**race condition**_.

### Race conditions
> A race condition occurs when the correctness of a computation depends on the relative timing or interleaving of multiple threads by the runtime; in other words, when getting the right answer relies on lucky timing.

The most common type of race condition is _check-then-act_, where a potentially stale observation is used to make a decision on what to do next. You observe something to be true (file X doesn’t exist) and then take action based on that observation (create X); but in fact the observation could have become invalid between the time you observed it and the time you acted on it (someone else created X in the meantime), causing a problem (unexpected exception, overwritten data, file corruption).

#### Example: race conditions in lazy initialization

A common idiom that uses _check-then-act_ is lazy initialization. The goal of lazy initialization is to defer initializing an object until it is actually needed while at the same time ensuring that it is initialized only once.

```{java}
@NotThreadSafe
public class LazyInitRace {
  private ExpensiveObject instance = null;
  
  public ExpensiveObject getInstance() { 
    if (instance == null) {
      instance = new ExpensiveObject(); 
      return instance;
    }
  } 
}
```

Now we have identified two sorts of race condition operations:

- _**Read-modify-write:**_ like incrementing a counter (i.e.: `count++;`), define a transformation of an object’s state in terms of its previous state.

- _**Check-then-act:**_ Check for a condition (i.e.: `if (instance == null)`) and then act (i.e.: `{instance = new ExpensiveObject(); ...}`)

Like most concurrency errors, race conditions don’t always result in failure: some unlucky timing is also required. But race conditions can cause serious problems.

#### Compound actions
> We refer collectively to check-then-act and read-modify-write sequences as compound actions: sequences of operations that must be executed atomically in order to remain thread-safe.

#### Fixing `UnsafeCountingFactorizer`
```{java}
@ThreadSafe
public class CountingFactorizer implements Servlet {
  private final AtomicLong count = new AtomicLong(0);
  
    public long getCount() { return count.get(); }
    
    public void service(ServletRequest req, ServletResponse resp) { 
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      count.incrementAndGet();
      encodeIntoResponse(resp, factors); 
    }
}
```
The `java.util.concurrent.atomic` package contains atomic variable classes for effecting atomic state transitions on numbers and object references. By replacing the long counter with an AtomicLong, we ensure that all actions that access the counter state are atomic. Because the state of the servlet is the state of the counter and the counter is thread-safe, our servlet is once again thread-safe.

## Locking

> To preserve state consistency, update related state variables in a single atomic operation.

### Intrinsic locks

Java provides a built-in locking mechanism for enforcing atomicity: the synchronized block. A synchronized block has two parts: a reference to an object that will serve as the lock, and a block of code to be guarded by that lock. A synchronized method is a shorthand for a synchronized block that spans an entire method body, and whose lock is the object on which the method is being invoked. (Static synchronized methods use the Class object for the lock.)

```{java}
synchronized (lock) {
  // Access or modify shared state guarded by lock
}
```

_**Every Java object can implicitly act as a lock for purposes of synchronization; these built-in locks are called intrinsic locks or monitor locks.**_ The lock is automatically acquired by the executing thread before entering a synchronized block and automatically released when control exits the synchronized block, whether by the normal control path or by throwing an exception out of the block. The only way to acquire an intrinsic lock is to enter a synchronized block or method guarded by that lock.

Intrinsic locks in Java act as mutexes (or mutual exclusion locks), which means that at most one thread may own the lock. When thread A attempts to acquire a lock held by thread B, A must wait, or block, until B releases it. If B never releases the lock, A waits forever.

No thread executing a synchronized block can observe another thread to be in the middle of a synchronized block guarded by the same lock. That is, the synchronized blocks guarded by the same lock execute atomically with respect to one another. In the context of concurrency, atomicity means the same thing as it does in transactional applications—that a group of statements appear to execute as a single, indivisible unit.

### Reentrancy

When a thread requests a lock that is already held by another thread, the requesting thread blocks. But _**because intrinsic locks are reentrant, if a thread tries to acquire a lock that it already holds, the request succeeds.**_ Reentrancy means that locks are acquired on a per-thread rather than per-invocation basis. Reentrancy facilitates encapsulation of locking behavior, and thus simplifies the development of object-oriented concurrent code.

```{java}
public class Widget {
  public synchronized void doSomething() {
    ... 
  }
}

public class LoggingWidget extends Widget { 
  public synchronized void doSomething() {
    System.out.println(toString() + ": calling doSomething");
    super.doSomething();
  }
}
```

Without reentrant locks, the very natural-looking code above, in which a subclass overrides a synchronized method and then calls the superclass method, would deadlock. Because the `doSomething` methods in `Widget` and `LoggingWidget` are both synchronized, each tries to acquire the lock on the `Widget` before proceeding. But if intrinsic locks were not reentrant, the call to super.doSomething would never be able to acquire the lock because it would be considered already held, and the thread would permanently stall waiting for a lock it can never acquire. Reentrancy saves us from deadlock in situations like this.

## Guarding state with locks
Because locks enable serialized[^1] access to the code paths they guard, we can use them to construct protocols for guaranteeing exclusive access to shared state. Following these protocols consistently can ensure state consistency.

Compound actions on shared state, such as incrementing a hit counter (_read-modify-write_) or lazy initialization (_check-then-act_), must be made atomic to avoid race conditions. Holding a lock for the entire duration of a compound action can make that compound action atomic. However, just wrapping the compound action with a synchronized block is not sufficient; if synchronization is used to coordinate access to a variable, it is needed everywhere that variable is accessed. Further, when using locks to coordinate access to a variable, the same lock must be used wherever that variable is accessed.

It is a common mistake to assume that synchronization needs to be used only when writing to shared variables; this is simply not true.

> For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock.

Acquiring the lock associated with an object does not prevent other threads from accessing that object—the only thing that acquiring a lock prevents any other thread from doing is acquiring that same lock. The fact that every object has a built-in lock is just a convenience so that you needn’t explicitly create lock objects[^2].

> Every shared, mutable variable should be guarded by exactly one lock. Make it clear to maintainers which lock that is.

A common locking convention is to encapsulate all mutable state within an object and to protect it from concurrent access by synchronizing any code path that accesses mutable state using the object’s intrinsic lock. Not all data needs to be guarded by locks—only mutable data that will be accessed from multiple threads. When a variable is guarded by a lock—meaning that every access to that variable is performed with that lock held—you’ve ensured that only one thread at a time can access that variable. When a class has invariants that involve more than one state variable, there is an additional requirement: each variable participating in the invariant must be guarded by the same lock. This allows you to access or update them in a single atomic operation, preserving the invariant.

> For every invariant that involves more than one variable, all the variables involved in that invariant must be guarded by the same lock.

If synchronization is the cure for race conditions, why not just declare ev- ery method synchronized?

```{java}
if (!vector.contains(element)) 
  vector.add(element);
```

This attempt at a put-if-absent operation has a race condition, even though both contains and add are atomic. While synchronized methods can make individual operations atomic, additional locking is required when multiple operations are combined into a compound action. At the same time, synchronizing every method can lead to liveness or performance problems.

## Liveness and performance

Deciding how big or small to make synchronized blocks may require tradeoffs among competing design forces, including safety (which must not be compromised), simplicity, and performance. Sometimes simplicity and performance are at odds with each other, a reasonable balance can usually be found.

Whenever you use locking, you should be aware of what the code in the block is doing and how likely it is to take a long time to execute. Holding a lock for a long time, either because you are doing something compute-intensive or because you execute a potentially blocking operation, introduces the risk of liveness or performance problems.

> Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network or console I/O.

[^1]: Serializing access to an object has nothing to do with object serialization (turning an object into a byte stream); serializing access means that threads take turns accessing the object exclusively, rather than doing so concurrently.

[^2]: In retrospect, this design decision was probably a bad one: not only can it be confusing, but it forces JVM implementors to make tradeoffs between object size and locking performance.