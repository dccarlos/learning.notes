# 4. Composing objects

## Designing a thread-safe class

> An invariant is a set of assertions that must always hold true during the life of an object for the program to be valid. A postcondition is a condition or predicate that must always be true just after the execution of some section of code or after an operation in a formal specification. Postconditions are sometimes tested using assertions within the code itself.

The design process for a thread-safe class should include these three basic elements:

- Identify the variables that form the object’s state
- Identify the invariants that constrain the state variables
- Establish a policy for managing concurrent access to the object’s
state.

An object’s state starts with its fields. If they are all of primitive type, the fields comprise the entire state. The state of an object with n primitive fields is just the n-tuple of its field values; the state of a 2D Point is its (x, y) value. _**If the object has fields that are references to other objects, its state will encompass fields from the referenced objects as well.**_

The _**synchronization policy**_ _defines how an object coordinates access to its state without violating its invariants or postconditions._ It specifies what combination of immutability, thread confinement, and locking is used to maintain thread safety, and which variables are guarded by which locks. _**To ensure that the class can be analyzed and maintained, document the synchronization policy.**_

### Gathering synchronization requirements

Making a class thread-safe means ensuring that its invariants hold under concurrent access; this requires reasoning about its state. Objects and variables have a state space: the range of possible states they can take on. The smaller this state space, the easier it is to reason about. Many classes have invariants that identify certain states as valid or invalid (i.e.: in counters negative values are not allowed).
Similarly, operations may have postconditions that identify certain state transi- tions as invalid. If the current state of a Counter is 17, the only valid next state is 18. _When the next state is derived from the current state, the operation is necessarily a compound action._
_**Constraints placed on states or state transitions by invariants and postconditions create additional synchronization or encapsulation requirements.**_

A class can also have invariants that constrain multiple state variables. _**Multivariable invariants like this one create atomicity requirements: related variables must be fetched or updated in a single atomic operation. You cannot update one, release and reacquire the lock, and then update the others, since this could involve leaving the object in an invalid state when the lock was released.**_

> You cannot ensure thread safety without understanding an object’s invariants and postconditions. Constraints on the valid values or state transitions for state variables can create atomicity and encapsulation requirements.

### State-dependent operations (i.e.: wait until true)

Class invariants and method postconditions constrain the valid states and state transitions for an object. Some objects also have methods with state-based preconditions. For example, you cannot remove an item from an empty queue; a queue must be in the “nonempty” state before you can remove an element. _**Operations with state-based preconditions are called state-dependent.**_

In a single-threaded program, if a precondition does not hold, the operation has no choice but to fail. But in a concurrent program, the precondition may become true later due to the action of another thread. Concurrent programs add the possibility of waiting until the precondition becomes true, and then proceeding with the operation.

The built-in mechanisms for efficiently waiting for a condition to become true—wait and notify—are tightly bound to intrinsic locking, and can be difficult to use correctly. To create operations that wait for a precondition to become true before proceeding, it is often easier to use existing library classes, such as blocking queues or semaphores, to provide the desired state-dependent behavior.

### State ownership

When defining which variables form an object’s state, we want to consider only the data that object owns. Ownership is not embodied explicitly in the language, but is instead an element of class design.

For better or worse, garbage collection lets us avoid thinking carefully about ownership. When passing an object to a method in C++, you have to think fairly carefully about whether you are transferring ownership, engaging in a short-term loan, or envisioning long-term joint ownership. In Java, all these same ownership models are possible, but the garbage collector reduces the cost of many of the common errors in reference sharing, enabling less-than-precise thinking about ownership.

In many cases, ownership and encapsulation go together—the object encapsulates the state it owns and owns the state it encapsulates. It is the owner of a given state variable that gets to decide on the locking protocol used to maintain the integrity of that variable’s state. Ownership implies control, but once you publish a reference to a mutable object, you no longer have exclusive control; at best, you might have “shared ownership”. A class usually does not own the objects passed to its methods or constructors, unless the method is designed to explicitly transfer ownership of objects passed in (such as the synchronized collection wrapper factory methods).

Collection classes often exhibit a form of “split ownership”, in which the collection owns the state of the collection infrastructure, but client code owns the objects stored in the collection.

## Instance confinement

If an object is not thread-safe you can ensure that it is only accessed from a single thread (thread confinement), or that all access to it is properly guarded by a lock.

Encapsulation simplifies making classes thread-safe by promoting instance confinement. When an object is encapsulated within another object, all code paths that have access to the encapsulated object are known and can be therefore be analyzed more easily than if that object were accessible to the entire program. Combining confinement with an appropriate locking discipline can ensure that otherwise non-thread-safe objects are used in a thread-safe manner.

Confined objects must not escape their intended scope. An object may be confined to a class instance (such as a private class member), a lexical scope (such as a local variable), or a thread (such as an object that is passed from method to method within a thread, but not supposed to be shared across threads).

`PersonSet` in code bellow illustrates how confinement and locking can work together to make a class thread-safe even when its component state variables are not. The state of `PersonSet` is managed by a HashSet, which is not thread-safe. But because mySet is private and not allowed to escape, the HashSet is confined to the PersonSet. The only code paths that can access mySet are addPerson and containsPerson, and each of these acquires the lock on the PersonSet. All its state is guarded by its intrinsic lock, making PersonSet thread-safe.

```{java}
@ThreadSafe
public class PersonSet {
  @GuardedBy("this") private final Set<Person> mySet = new HashSet<Person>();
  public synchronized void addPerson(Person p) { 
    mySet.add(p);
  }

  public synchronized boolean containsPerson(Person p) { 
    return mySet.contains(p);
  } 
}
```

Above example makes no assumptions about the thread-safety of `Person`, but if it is mutable, additional synchronization will be needed when accessing a Person retrieved from a `PersonSet`. The most reliable way to do this would be to make `Person` thread-safe; less reliable would be to guard the `Person` objects with a lock and ensure that all clients follow the protocol of acquiring the appropriate lock before accessing the `Person`.

Of course, it is still possible to violate confinement by publishing a supposedly confined object; if an object is intended to be confined to a specific scope, then letting it escape from that scope is a bug. Confined objects can also escape by publishing other objects such as iterators or inner class instances that may indirectly publish the confined objects.

### The Java monitor pattern

Following the principle of instance confinement to its logical conclusion leads you to the Java monitor pattern. An object following the Java monitor pattern encapsulates all its mutable state and guards it with the object’s own intrinsic lock.

The Java monitor pattern is merely a convention; any lock object could be used to guard an object’s state so long as it is used consistently.

```{java}
public class PrivateLock {
  private final Object myLock = new Object(); 
  @GuardedBy("myLock") Widget widget;

  void someMethod() { 
    synchronized(myLock) {
      // Access or modify the state of widget
    } 
  }
}
```

_**There are advantages to using a private lock object instead of an object’s intrinsic lock (or any other publicly accessible lock). Making the lock object private encapsulates the lock so that client code cannot acquire it, whereas a publicly accessible lock allows client code to participate in its synchronization policy—correctly or incorrectly. Clients that improperly acquire another object’s lock could cause liveness problems, and verifying that a publicly accessible lock is properly used requires examining the entire program rather than a single class.**_

## Delegating thread safety

The Java monitor pattern is useful when building classes from scratch or composing classes out of objects that are not thread-safe. But what if the components of our class are already thread-safe? Do we need to add an additional layer of thread safety? The answer is: “it depends”.

In `CountingFactorizer`, we added an AtomicLong to an otherwise stateless object, and the resulting composite object was still thread-safe. Since the state of `CountingFactorizer` is the state of the thread-safe AtomicLong, and since CountingFactorizer imposes no additional validity constraints on the state of the counter, it is easy to see that `CountingFactorizer` is thread-safe. We could say that `CountingFactorizer` delegates its thread safety responsibilities to the AtomicLong: `CountingFactorizer` is thread-safe because `AtomicLong` is.

### Independent state variables

We can also delegate thread safety to more than one underlying state variable as long as those underlying state variables are independent, meaning that the composite class does not impose any invariants involving the multiple state variables.

### When delegation fails

`NumberRange` in bellow code uses two AtomicIntegers to manage its state, but imposes an additional constraint—that the first number be less than or equal to the second.

```{java}
public class NumberRange {
  // INVARIANT: lower <= upper
  private final AtomicInteger lower = new AtomicInteger(0); 
  private final AtomicInteger upper = new AtomicInteger(0);
  
  public void setLower(int i) {
    // Warning -- unsafe check-then-act
    if (i > upper.get())
      throw new IllegalArgumentException("can’t set lower to " + i + " > upper"); 
      lower.set(i);
    }
  }
  
  public void setUpper(int i) {
    // Warning -- unsafe check-then-act
    if (i < lower.get())
      throw new IllegalArgumentException("can’t set upper to " + i + " < lower"); 
      upper.set(i);
  }
  
  public boolean isInRange(int i) {
    return (i >= lower.get() && i <= upper.get());
  }
} 
```

`NumberRange` is not thread-safe; it does not preserve the invariant that constrains lower and upper. The setLower and setUpper methods attempt to respect this invariant, but do so poorly. Both setLower and setUpper are check-then-act sequences, but they do not use sufficient locking to make them atomic. If the number range holds (0, 10), and one thread calls setLower(5) while another thread calls setUpper(4), with some unlucky timing both will pass the checks in the setters and both modifications will be applied. The result is that the range now holds (5, 4)—an invalid state. So while the underlying `AtomicIntegers` are thread-safe, the composite class is not. Because the underlying state variables lower and upper are not independent, `NumberRange` cannot simply delegate thread safety to its thread-safe state variables. `NumberRange` could be made thread-safe by using locking to maintain its in- variants, such as guarding lower and upper with a common lock.

If a class has compound actions, as `NumberRange` does, delegation alone is again not a suitable approach for thread safety. In these cases, the class must provide its own locking to ensure that compound actions are atomic, unless the entire compound action can also be delegated to the underlying state variables.

### Publishing underlying state variables

When you delegate thread safety to an object’s underlying state variables, under what conditions can you publish those variables so that other classes can modify them as well? Again, the answer depends on what invariants your class imposes on those variables.

> If a state variable is thread-safe, does not participate in any invariants that constrain its value, and has no prohibited state transitions for any of its operations, then it can safely be published.

## Adding functionality to existing thread-safe classes

Sometimes a thread-safe class that supports all of the operations we want already exists, but often the best we can find is a class that supports almost all the operations we want, and then we need to add a new operation to it without undermining its thread safety.

The safest way to add a new atomic operation is to modify the original class to support the desired operation, but this is not always possible because you may not have access to the source code or may not be free to modify it. If you can modify the original class, you need to understand the implementation’s synchronization policy so that you can enhance it in a manner consistent with its original design. Adding the new method directly to the class means that all the code that implements the synchronization policy for that class is still contained in one source file, facilitating easier comprehension and maintenance.

Another approach is to extend the class, assuming it was designed for extension.

Extension is more fragile than adding code directly to a class, because the implementation of the synchronization policy is now distributed over multiple, separately maintained source files. If the underlying class were to change its synchronization policy by choosing a different lock to guard its state variables, the subclass would subtly and silently break, because it no longer used the right lock to control concurrent access to the base class’s state.

### Client-side locking

A third strategy is to extend the functionality of the class without extending the class itself by placing extension code in a “helper” class.

```{java}
@NotThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<E>());
  // ...
  public synchronized boolean putIfAbsent(E x) {
    boolean absent = !list.contains(x);
    
    if (absent)
      list.add(x);
    return absent;
  }
}  
```

Why wouldn’t this work? After all, `putIfAbsent` is synchronized, right? The problem is that it synchronizes on the wrong lock. Whatever lock the List uses to guard its state, it sure isn’t the lock on the `ListHelper`. `ListHelper` provides only the illusion of synchronization; the various list operations, while all synchronized, use different locks, which means that `putIfAbsent` is not atomic relative to other operations on the List. So there is no guarantee that another thread won’t modify the list while `putIfAbsent` is executing.

To make this approach work, we have to use the same lock that the `List` uses by using client-side locking or external locking. Client-side locking implies guarding client code that uses some object X with the lock X uses to guard its own state. In order to use client-side locking, you must know what lock X uses.

The documentation for `Vector` and the synchronized wrapper classes states, not clearly, that they support client-side locking, by using the intrinsic lock for the Vector or the wrapper collection (not the wrapped collection). Bellow code shows a putIfAbsent operation on a thread-safe List that correctly uses client-side locking.

```{java}
@ThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<E>());
  // ...
  public synchronized boolean putIfAbsent(E x) {
    synchronized (list) {
      boolean absent = !list.contains(x);
    
      if (absent)
        list.add(x);
      return absent;
    }
  }
}  
```

If extending a class to add another atomic operation is fragile because it distributes the locking code for a class over multiple classes in an object hierarchy, client-side locking is even more fragile because it entails putting locking code for class C into classes that are totally unrelated to C. Exercise care when using client-side locking on classes that do not commit to their locking strategy.

Client-side locking has a lot in common with class extension—they both couple the behavior of the derived class to the implementation of the base class. Just as extension violates encapsulation of implementation, client-side locking violates encapsulation of synchronization policy.

### Composition

There is a less fragile alternative for adding an atomic operation to an existing class: composition.
`ImprovedList` in bellow code implements the List operations by delegating them to an underlying List instance, and adds an atomic `putIfAbsent` method. (Like `Collections.synchronizedList` and other collections wrappers, `ImprovedList` assumes that once a list is passed to its constructor, the client will not use the underlying list directly again, accessing it only through the `ImprovedList`.)

```{java}
@ThreadSafe
public class ImprovedList<T> implements List<T> {
  private final List<T> list;
  public ImprovedList(List<T> list) { this.list = list; }
  
  public synchronized boolean putIfAbsent(T x) { 
    boolean contains = list.contains(x);
    
    if (contains)
      list.add(x);
    return !contains;
  }
  
  public synchronized void clear() { 
    list.clear(); 
  }
  // ... similarly delegate other List methods
}
```

`ImprovedList` adds an additional level of locking using its own intrinsic lock. It does not care whether the underlying List is thread-safe, because it provides its own consistent locking that provides thread safety even if the List is not thread-safe or changes its locking implementation. While the extra layer of synchronization may add some small performance penalty, the implementation in `ImprovedList` is less fragile than attempting to mimic the locking strategy of another object. In effect, we’ve used the Java monitor pattern to encapsulate an existing List, and this is guaranteed to provide thread safety so long as our class holds the only outstanding reference to the underlying List.

## Documenting synchronization policies

Documentation is one of the most powerful (and, sadly, most underutilized) tools for managing thread safety. Users look to the documentation to find out if a class is thread-safe, and maintainers look to the documentation to understand the implementation strategy so they can maintain it without inadvertently compromising safety. Unfortunately, both of these constituencies usually find less information in the documentation than they’d like.

> Document a class’s thread safety guarantees for its clients; document its synchronization policy for its maintainers.

Each use of synchronized, volatile, or any thread-safe class reflects a synchronization policy defining a strategy for ensuring the integrity of data in the face of concurrent access. That policy is an element of your program’s design, and should be documented. Of course, the best time to document design decisions is at design time. Weeks or months later, the details may be a blur—so write it down before you forget.

### Interpreting vague documentation

“thread” and “concurrent” do not appear at all in the `JDBC` specification, and appear frustratingly rarely in the servlet specification. So what do you do?

You are going to have to guess. One way to improve the quality of your guess is to interpret the specification from the perspective of someone who will implement it (such as a container or database vendor), as opposed to someone who will merely use it. Servlets are always called from a container-managed thread, and it is safe to assume that if there is more than one such thread, the container knows this.