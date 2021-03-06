# Summary of Part I

- _**It’s the mutable state, stupid.**_ All concurrency issues boil down to coordinating access to mutable state. The less mutable state, the easier it is to ensure thread safety.
- _**Make fields final unless they need to be mutable.**_
- _**Immutable objects are automatically thread-safe.**_ Immutable objects simplify concurrent programming tremendously. They are simpler and safer, and can be shared freely without locking or defensive copying.
- _**Encapsulation makes it practical to manage the complexity.**_ You could write a thread-safe program with all data stored in global variables, but why would you want to? Encapsulating data within objects makes it easier to preserve their invariants; encapsulating synchronization within objects makes it easier to comply with their synchronization policy.
- _**Guard each mutable variable with a lock.**_
- _**Guard all variables in an invariant with the same lock.**_
- _**Hold locks for the duration of compound actions.**_
- _**A program that accesses a mutable variable from multiple threads without synchronization is a broken program.**_
- _**Don’t rely on clever reasoning about why you don’t need to synchronize.**_
- _**Include thread safety in the design process—or explicitly document that your class is not thread-safe.**_
- _**Document your synchronization policy.**_

> Review `volatile` and `ThreadLocal` concepts (again)