# 5. Building Blocks

Where practical, delegation is one of the most effective strategies for creating thread-safe classes: just let existing thread-safe classes manage all the state.

## Synchronized collections
### Problems with synchronized collections

The synchronized collections are thread-safe, but you may sometimes need to use additional client-side locking to guard compound actions.

Because the synchronized collections commit to a synchronization policy that supports client-side locking, it is possible to create new operations that are atomic with respect to other collection operations as long as we know which lock to use. The synchronized collection classes guard each method with the lock on the synchronized collection object itself.

The problem of unreliable iteration can again be addressed by client-side locking, at some additional cost to scalability. By holding the Vector lock for the duration of iteration, we prevent other threads from modifying the Vector while we are iterating it. Unfortunately, we also prevent other threads from accessing it at all during this time, impairing concurrency.

### Iterators and `ConcurrentModificationException`

The standard way to iterate a `Collection` is with an `Iterator`, either explicitly or through the for-each loop syntax introduced in Java 5.0, but using iterators does not obviate the need to lock the collection during iteration if other threads can concurrently modify it.

The iterators returned by the synchronized collections are not designed to deal with concurrent modification, and they are fail-fast—meaning that if they detect that the collection has changed since iteration began, they throw the unchecked `ConcurrentModificationException`.

These fail-fast iterators are not designed to be foolproof—they are designed to catch concurrency errors on a “good-faith-effort” basis and thus act only as early-warning indicators for concurrency problems. They are implemented by associating a modification count with the collection: if the modification count changes during iteration, `hasNext` or `next` throws `ConcurrentModificationException`. However, this check is done without synchronization, so there is a risk of seeing a stale value of the modification count and therefore that the iterator does not realize a modification has been made. This was a deliberate design tradeoff to reduce the performance impact of the concurrent modification detection code[^1].

```{java}
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
// ...
// May throw ConcurrentModificationException
for (Widget w : widgetList) 
  doSomething(w);
```

There are several reasons, however, why locking a collection during iteration may be undesirable. Other threads that need to access the collection will block until the iteration is complete; if the collection is large or the task performed for each element is lengthy, they could wait a long time. Also, if the collection is locked as above listing, `doSomething` is being called with a lock held, which is a risk factor for deadlock (see Chapter 10). Even in the absence of starvation or deadlock risk, locking collections for significant periods of time hurts application scalability. The longer a lock is held, the more likely it is to be contended, and if many threads are blocked waiting for a lock throughput and CPU utilization can suffer.

An alternative to locking the collection during iteration is to clone the collection and iterate the copy instead. Since the clone is thread-confined, no other thread can modify it during iteration, eliminating the possibility of `ConcurrentModificationException`. (The collection still must be locked during the clone operation itself.) Cloning the collection has an obvious performance cost; whether this is a favorable tradeoff depends on many factors including the size of the collection, how much work is done for each element, the relative frequency of iteration compared to other collection operations, and responsiveness and throughput requirements.

### Hidden iterators

While locking can prevent iterators from throwing `ConcurrentModificationException`, you have to remember to use locking everywhere a shared collection might be iterated. This is trickier than it sounds, as iterators are sometimes hidden (like in `toString`).

```{java}
private final Set<Integer> mySet = new HashSet<Integer>();
public void addTenThings() {
  System.out.println("DEBUG: added ten elements to " + set);
}
```

The real lesson here is that the greater the distance between the state and the synchronization that guards it, the more likely that someone will forget to use proper synchronization when accessing that state. By using a `synchronizedSet`, encapsulating the synchronization, this sort of error would not occur.

> Just as encapsulating an object’s state makes it easier to preserve its invariants, encapsulating its synchronization makes it easier to enforce its synchronization policy.

Iteration is also indirectly invoked by the collection’s `hashCode` and `equals` methods, which may be called if the collection is used as an element or key of another collection. Similarly, the `containsAll`, `removeAll`, and `retainAll` methods, as well as the constructors that take collections as arguments, also iterate the collection. All of these indirect uses of iteration can cause `ConcurrentModificationException`.

## Concurrent collections

Java 5.0 improves on the synchronized collections by providing several concurrent collection classes. Synchronized collections achieve their thread safety by serializing all access to the collection’s state. The cost of this approach is poor concurrency; when multiple threads contend for the collection-wide lock, throughput suffers.

The concurrent collections, on the other hand, are designed for concurrent access from multiple threads. Java 5.0 adds `ConcurrentHashMap`, a replacement for synchronized hash-based `Map` implementations, and `CopyOnWriteArrayList`, a replacement for synchronized `List` implementations for cases where traversal is the dominant operation. The new `ConcurrentMap` interface adds support for common compound actions such as put-if-absent, replace, and conditional remove.

> Replacing synchronized collections with concurrent collections can offer dramatic scalability improvements with little risk.

Java 5.0 also adds two new collection types, `Queue` and `BlockingQueue`. A `Queue` is intended to hold a set of elements temporarily while they await processing. Several implementations are provided, including `ConcurrentLinkedQueue`, a traditional FIFO queue, and `PriorityQueue`, a (non concurrent) priority ordered queue. Queue classes were added because eliminating the random-access requirements of List admits more efficient concurrent implementations.

`BlockingQueue` extends `Queue` to add blocking insertion and retrieval operations. If the queue is empty, a retrieval blocks until an element is available, and if the queue is full (for bounded queues) an insertion blocks until there is space available. Blocking queues are extremely useful in producer-consumer designs.

Just as `ConcurrentHashMap` is a concurrent replacement for a synchronized hash-based Map, Java 6 adds `ConcurrentSkipListMap` and `ConcurrentSkipListSet`, which are concurrent replacements for a synchronized `SortedMap` or `SortedSet` (such as `TreeMap` or `TreeSet` wrapped with `synchronizedMap`).

### `ConcurrentHashMap`

`ConcurrentHashMap` is a hash-based Map like HashMap, but it uses an entirely different locking strategy that offers better concurrency and scalability. Instead of synchronizing every method on a common lock, restricting access to a single thread at a time, it uses a finer-grained locking mechanism called _lock striping_ to allow a greater degree of shared access. Arbitrarily many reading threads can access the map concurrently, readers can access the map concurrently with writers, and _**a limited number of writers can modify the map concurrently.**_ The result is far higher throughput under concurrent access, with little performance penalty for single-threaded access.

`ConcurrentHashMap`, along with the other concurrent collections, further improve on the synchronized collection classes by providing iterators that do not throw `ConcurrentModificationException`, thus eliminating the need to lock the collection during iteration. The iterators returned by `ConcurrentHashMap` are weakly consistent instead of fail-fast. A weakly consistent iterator can tolerate concurrent modification, traverses elements as they existed when the iterator was constructed, and may (but is not guaranteed to) reflect modifications to the collection after the construction of the iterator.

As with all improvements, there are still a few tradeoffs: `size` is allowed to return an approximation instead of an exact count or `isEmpty` may not reflect if the structure is empty. So the requirements for these operations were weakened to enable performance optimizations for the most important operations, primarily `get`, `put`, `containsKey`, and `remove`.

> Because it has so many advantages and so few disadvantages compared to `Hashtable` or `synchronizedMap`, replacing synchronized Map implementations with `ConcurrentHashMap` in most cases results only in better scalability. Only if your application needs to lock the map for exclusive access is `ConcurrentHashMap` not an appropriate drop-in replacement.

> Since a `ConcurrentHashMap` cannot be locked for exclusive access, we cannot use client-side locking to create new atomic operations such as put-if-absent but it already have those implemented.

### `CopyOnWriteArrayList`

`CopyOnWriteArrayList` is a concurrent replacement for a synchronized `List` that offers better concurrency in some common situations and eliminates the need to lock or copy the collection during iteration.

The copy-on-write collections derive their thread safety from the fact that as long as an effectively immutable object is properly published, no further synchronization is required when accessing it. They implement mutability by creating and republishing a new copy of the collection every time it is modified.

Iterators for the copy-on-write collections retain a reference to the backing array that was current at the start of iteration, and since this will never change, they need to synchronize only briefly to ensure visibility of the array contents. As a result, multiple threads can iterate the collection without interference from one another or from threads wanting to modify the collection. The iterators returned by the copy-on-write collections do not throw `ConcurrentModificationException` and return the elements exactly as they were at the time the iterator was created, regardless of subsequent modifications.

## Blocking queues and the producer-consumer pattern

Blocking queues provide blocking `put` and `take` methods as well as the timed equivalents `offer` and `poll`. If the queue is full, `put` blocks until space becomes available; if the queue is empty, `take` blocks until an element is available. Queues can be bounded or unbounded; unbounded queues are never full, so a `put` on an unbounded queue never blocks.

The familiar division of labor for two people washing the dishes is an example of a producer-consumer design: one person washes the dishes and places them in the dish rack, and the other person retrieves the dishes from the rack and dries them. In this scenario, the dish rack acts as a blocking queue; if there are no dishes in the rack, the consumer waits until there are dishes to dry, and if the rack fills up, the producer has to stop washing until there is more space.

The last `BlockingQueue` implementation, `SynchronousQueue`, is not really a queue at all, in that it maintains no storage space for queued elements. Instead, it maintains a list of queued threads waiting to enqueue or dequeue an element. In the dish-washing analogy, this would be like having no dish rack, but instead handing the washed dishes directly to the next available dryer. While this may seem a strange way to implement a queue, it reduces the latency associated with moving data from producer to consumer because the work can be handed off directly. Since a `SynchronousQueue` has no storage capacity, put and take will block unless another thread is already waiting to participate in the handoff. Synchronous queues are generally suitable only when there are enough consumers that there nearly always will be one ready to take the handoff.

### Serial thread confinement

For mutable objects, producer-consumer designs and blocking queues facilitate serial thread confinement for handing off ownership of objects from producers to consumers. A thread-confined object is owned exclusively by a single thread, but that ownership can be “transferred” by publishing it safely where only one other thread will gain access to it and ensuring that the publishing thread does not access it after the handoff. The safe publication ensures that the object’s state is visible to the new owner, and since the original owner will not touch it again, it is now confined to the new thread. The new owner may modify it freely since it has exclusive access.

### Deques and work stealing

Java 6 also adds another two collection types, `Deque` (pronounced “deck”) and `BlockingDeque`, that extend `Queue` and `BlockingQueue`. A `Deque` is a double-ended queue that allows efficient insertion and removal from both the head and the tail. Implementations include `ArrayDeque` and `LinkedBlockingDeque`.

Just as blocking queues lend themselves to the producer-consumer pattern, deques lend themselves to a related pattern called work stealing. A producer-consumer design has one shared work queue for all consumers; in a work stealing design, every consumer has its own deque. If a consumer exhausts the work in its own deque, it can steal work from the tail of someone else’s deque.

## Blocking and interruptible methods

The distinction between a blocking operation and an ordinary operation that merely takes a long time to finish is that a blocked thread must wait for an event that is beyond its control before it can proceed—the I/O completes, the lock becomes available, or the external computation finishes. When that external event occurs, the thread is placed back in the `RUNNABLE` state and becomes eligible again for scheduling.

The `put` and `take` methods of `BlockingQueue` throw the checked `InterruptedException`, as do a number of other library methods such as `Thread.sleep`. When a method can throw InterruptedException, it is telling you that it is a blocking method, and further that if it is interrupted, it will make an effort to stop blocking early.

`Thread` provides the interrupt method for interrupting a thread and for querying whether a thread has been interrupted. Each thread has a boolean property that represents its interrupted status; interrupting a thread sets this status.

Interruption is a cooperative mechanism. One thread cannot force another to stop what it is doing and do something else; when thread A interrupts thread B, A is merely requesting that B stop what it is doing when it gets to a convenient stopping point—if it feels like it. While there is nothing in the API or language specification that demands any specific application-level semantics for interruption, the most sensible use for interruption is to cancel an activity. Blocking methods that are responsive to interruption make it easier to cancel long-running activities on a timely basis.

_**When your code calls a method that throws `InterruptedException`, then your method is a blocking method too, and must have a plan for responding to interruption.**_ For library code, there are basically two choices:

- _**Propagate the InterruptedException.**_ This is often the most sensible policy if you can get away with it—just propagate the `InterruptedException` to your caller. This could involve not catching InterruptedException, or catching it and throwing it again after performing some brief activity-specific cleanup.

- _**Restore the interrupt.**_ Sometimes you cannot throw InterruptedException, for instance when your code is part of a Runnable. In these situations, you must catch InterruptedException and restore the interrupted status by calling interrupt on the current thread, so that code higher up the call stack can see that an interrupt was issued.

```{java}
public class TaskRunnable implements Runnable { 
  BlockingQueue<Task> queue;
  ...
  public void run() {
    try { 
      processTask(queue.take());
    } catch (InterruptedException e) { 
      // restore interrupted status
      Thread.currentThread().interrupt();
    }
  } 
}
```

You can get much more sophisticated with interruption, but these two approaches should work in the vast majority of situations. But there is one thing you should not do with `InterruptedException`—catch it and do nothing in response.

> The only situation in which it is acceptable to swallow an interrupt is when you are extending Thread and therefore control all the code higher up on the call stack.

## Synchronizers

A synchronizer is any object that coordinates the control flow of threads based on its state. Blocking queues can act as synchronizers (they coordinate the control flow of producer and consumer threads because `take` and `put` block until the queue enters the desired state (not empty or not full)); other types of synchronizers include semaphores, barriers, and latches.

_**All synchronizers share certain structural properties: they encapsulate state that determines whether threads arriving at the synchronizer should be allowed to pass or forced to wait, provide methods to manipulate that state, and provide methods to wait efficiently for the synchronizer to enter the desired state.**_

### Latches

A latch is a synchronizer that can delay the progress of threads until it reaches its terminal state then it will allow threads to continue. Once the latch reaches the terminal state, it cannot change state again, so it remains open forever. _**Latches can be used to ensure that certain activities do not proceed until other one-time activities complete.**_

- Ensuring that a computation does not proceed until resources it needs have been initialized.
- Ensuring that a service does not start until other services on which it depends have started.
- Waiting until all the parties involved in an activity, for instance the players in a multi-player game, are ready to proceed.

#### **`CountDownLatch`**

It is a flexible latch implementation that can be used in any of these situations; it allows one or more threads to wait for a set of events to occur. The latch state consists of a counter initialized to a positive number, representing the number of events to wait for. The `countDown` method decrements the counter, indicating that an event has occurred, and the `await` methods wait for the counter to reach zero, which happens when all the events have occurred. If the counter is nonzero on entry, await blocks until the counter reaches zero, the waiting thread is interrupted, or the wait times out.

```{java}
public class TestHarness {
  public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
    final CountDownLatch startGate = new CountDownLatch(1); 
    final CountDownLatch endGate = new CountDownLatch(nThreads);
    
    for (int i = 0; i < nThreads; i++) {
      Thread t = new Thread() {
        public void run() {
          try {
            startGate.await();
            
            try {
              task.run();
            } finally {
              endGate.countDown();
            }
          } catch(InterruptedException ignored) {}
        }
      };
      
      t.start();
    }
    
    long start = System.nanoTime(); 
    startGate.countDown(); 
    endGate.await();
    long end = System.nanoTime(); 
    return end-start;
  }
}
```

`TestHarness` illustrates two common uses for latches. `TestHarness` creates a number of threads that run a given task concurrently. It uses two latches, a “starting gate” and an “ending gate”. The starting gate is initialized with a count of one; the ending gate is initialized with a count equal to the number of worker threads. The first thing each worker thread does is wait on the starting gate; this ensures that none of them starts working until they all are ready to start. The last thing each does is count down on the ending gate; this allows the master thread to wait efficiently until the last of the worker threads has finished, so it can calculate the elapsed time.

### **`FutureTask`**

`FutureTask` also acts like a latch. (`FutureTask` implements `Future`, which describes an abstract result-bearing computation.) A computation represented by a `FutureTask` is implemented with a `Callable`, the result-bearing equivalent of `Runnable`, and can be in one of three states: waiting to run, running, or completed. Completion subsumes all the ways a computation can complete, including normal completion, cancellation, and exception. Once a FutureTask enters the completed state, it stays in that state forever.

The behavior of `Future.get` depends on the state of the task. If it is completed, get returns the result immediately, and otherwise blocks until the task transitions to the completed state and then returns the result or throws an exception. `FutureTask` transports the result from the thread executing the computation to the thread(s) retrieving the result; the specification of `FutureTask` guarantees that this transfer constitutes a safe publication of the result.

> It is inadvisable to start a thread from a constructor or static initializer.

### Semaphores

Counting semaphores are used to control the number of activities that can access a certain resource or perform a given action at the same time. Counting semaphores can be used to implement resource pools or to impose a bound on a collection.

A Semaphore manages a set of virtual permits; the initial number of permits is passed to the Semaphore constructor. Activities can acquire permits (as long as some remain) and release permits when they are done with them. _**If no permit is available, `acquire` blocks until one is (or until interrupted or the operation times out).**_ The release method returns a permit to the semaphore[^2].

A degenerate case of a counting semaphore is a binary semaphore, a Semaphore with an initial count of one. A binary semaphore can be used as a mutex with nonreentrant locking semantics; whoever holds the sole permit holds the mutex.

Semaphores are useful for implementing resource pools such as database connection pools. While it is easy to construct a fixed-sized pool that fails if you request a resource from an empty pool, what you really want is to block if the pool is empty and unblock when it becomes nonempty again.

If you initialize a Semaphore to the pool size, `acquire` a permit before trying to fetch a resource from the pool, and `release` the permit after putting a resource back in the pool, `acquire` blocks until the pool becomes nonempty.

```{java}
public class BoundedHashSet<T> { 
  private final Set<T> set; 
  private final Semaphore sem;

  public BoundedHashSet(int bound) {
    this.set = Collections.synchronizedSet(new HashSet<T>()); 
    sem = new Semaphore(bound);
  }
  
  public boolean add(T o) throws InterruptedException { 
    sem.acquire();
    boolean wasAdded = false; 
    try {
      wasAdded = set.add(o);
      return wasAdded;
    }
    finally {
      if (!wasAdded)
        sem.release();
    } 
  }
  
  public boolean remove(Object o) { 
    boolean wasRemoved = set.remove(o); 
    if (wasRemoved)
      sem.release();
    return wasRemoved;
  }
}
```

The semaphore is initialized to the desired maximum size of the collection. The `add` operation acquires a permit before adding the item into the underlying collection. If the underlying `add` operation does not actually add anything, it releases the permit immediately. Similarly, a successful `remove` operation releases a permit, enabling more elements to be added. The underlying `Set` implementation knows nothing about the bound; this is handled by BoundedHashSet.

### Barriers

Barriers are similar to latches in that they block a group of threads until some event has occurred. The key difference is that with a barrier, all the threads must come together at a barrier point at the same time in order to proceed. Latches are for waiting for events; _**barriers are for waiting for other threads.**_

A barrier implements the protocol some families use to meet at an agreed time and place during a day at the mall: “Everyone meet at McDonald’s at 6:00; once you get there, stay there until everyone shows up, and then we’ll figure out what we’re doing next.”

#### `CyclicBarrier`

It allows a fixed number of parties to rendezvous repeatedly at a barrier point and is useful in parallel iterative algorithms that break down a problem into a fixed number of independent subproblems. _**Threads call await when they reach the barrier point, and await blocks until all the threads have reached the barrier point. If all threads meet at the barrier point, the barrier has been successfully passed, in which case all threads are released and the barrier is reset so it can be used again.**_

If a call to await times out or a thread blocked in await is interrupted, then the barrier is considered broken and all outstanding calls to await terminate with `BrokenBarrierException`. If the barrier is successfully passed, await returns a unique arrival index for each thread, which can be used to “elect” a leader that takes some special action in the next iteration. `CyclicBarrier` also lets you pass a barrier action to the constructor; this is a Runnable that is executed (in one of the subtask threads) when the barrier is successfully passed but before the blocked threads are released.

Another form of barrier is `Exchanger`, a two-party barrier in which the parties exchange data at the barrier point. Exchangers are useful when the parties perform asymmetric activities, for example when one thread fills a buffer with data and the other thread consumes the data from the buffer; these threads could use an `Exchanger` to meet and exchange a full buffer for an empty one. When two threads exchange objects via an Exchanger, the exchange constitutes a safe publication of both objects to the other party.

The timing of the exchange depends on the responsiveness requirements of the application. The simplest approach is that the filling task exchanges when the buffer is full, and the emptying task exchanges when the buffer is empty; this minimizes the number of exchanges but can delay processing of some data if the arrival rate of new data is unpredictable. Another approach would be that the filler exchanges when the buffer is full, but also when the buffer is partially filled and a certain amount of time has elapsed.

```{java}
public class CellularAutomata {
    private final Board mainBoard;
    private final CyclicBarrier barrier;
    private final Worker[] workers;

    public CellularAutomata(Board board) {
        this.mainBoard = board;
        int count = Runtime.getRuntime().availableProcessors();
        this.barrier = new CyclicBarrier(count,
                new Runnable() {
                    public void run() {
                        mainBoard.commitNewValues();
                    }});
        this.workers = new Worker[count];
        for (int i = 0; i < count; i++)
            workers[i] = new Worker(mainBoard.getSubBoard(count, i));
    }

    private class Worker implements Runnable {
        private final Board board;

        public Worker(Board board) { this.board = board; }
        public void run() {
            while (!board.hasConverged()) {
                for (int x = 0; x < board.getMaxX(); x++)
                    for (int y = 0; y < board.getMaxY(); y++)
                        board.setNewValue(x, y, computeValue(x, y));
                try {
                    barrier.await();
                } catch (InterruptedException ex) {
                    return;
                } catch (BrokenBarrierException ex) {
                    return;
                }
            }
        }

        private int computeValue(int x, int y) {
            // Compute the new value that goes in (x,y)
            return 0;
        }
    }

    public void start() {
        for (int i = 0; i < workers.length; i++)
            new Thread(workers[i]).start();
        mainBoard.waitForConvergence();
    }

    interface Board {
        int getMaxX();
        int getMaxY();
        int getValue(int x, int y);
        int setNewValue(int x, int y, int value);
        void commitNewValues();
        boolean hasConverged();
        void waitForConvergence();
        Board getSubBoard(int numPartitions, int index);
    }
}
```

`CellularAutomata` in above listing demonstrates using a barrier to compute a cellular automata simulation. When parallelizing a simulation, it is generally impractical to assign a separate thread to each element (in the case of Life, a cell); this would require too many threads, and the overhead of coordinating them would dwarf the computation. Instead, it makes sense to partition the problem into a number of subparts, let each thread solve a subpart, and then merge the results. `CellularAutomata` partitions the board into N-cpu parts, where N-cpu is the number of CPUs available, and assigns each part to a thread. At each step, the worker threads calculate new values for all the cells in their part of the board. When all worker threads have reached the barrier, the barrier action commits the new values to the data model. After the barrier action runs, the worker threads are released to compute the next step of the calculation, which includes consulting an isDone method to determine whether further iterations are required.





[^1]: `ConcurrentModificationException` can arise in single-threaded code as well; this happens when objects are removed from the collection directly rather than through `Iterator.remove`.

[^2]: The implementation has no actual permit objects, and Semaphore does not associate dispensed permits with threads, so a permit acquired in one thread can be released from another thread. You can think of `acquire` as consuming a permit and release as creating one; a Semaphore is not limited to the number of permits it was created with.