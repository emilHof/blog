# Road to Lockfree Priority Queue

### What is Lockfree

Among the different concurrency enabling paradigms, lockfree tries to be unique by offering a
way to operate on shared memory without the need for locks. In scenarios where lots of CPUs
are contending on inserts and deletions, acquiring locks can impose a significant increase
in runtime.

The idea of using a lockfree data structure to avoid the pitfalls of locks, is definitely not a
new one. Thanks to all the insanely smart scientists and engineers that have worked on this problem
we have a multitude of different implementations of mechanisms that enable lockfreedom.

It should also definitely be mentioned that it can often be debatable how much better lockfree
data structures really are. Much of this uncertainty could be due to how difficult it is to implement
lockfree code correctly and efficiently.

### Why a Skip List

While Binary Heaps are an extremely useful data structure for many algorithms, when it comes to concurrent
applications, they have some disadvantages that cannot be overlooked. They allow for an efficient
priority queue that supports the `insert()` and `remove_min()` operations to both complete in **O(log\*n)**
amortized time, yet these operations require a constant restoring of the heap's invariant by "bubbling"
[up](https://www.lavivienpost.net/max-heap-implementation/) or down nodes.

For concurrent applications [Skip List](https://www.mydistributed.systems/2021/03/skip-list-data-structure.html)
appear to be the winner. They are implemented similarly to a linked list, yet they hold additional properties,
such as **Forward Pointers**, that make searching them more efficient.

### Challenges

As my knowledge of Skip Lists is rather limited, glancing over one or two articles that is, I expect to take
quite a lot of time to learn about this data structure.

### Initial Plan

```rust
// How do we maintain the heap invariant?
pub struct LFHeap<K, V> {
    root: NonNull<HeapNode<K,V>>,
	gc: Arc<ShareGC<K,V>>,
    /* some data */
}

// Drop is implemented so to automatically restore the heap invariant
struct HeapNode<K,V> {
    key: K,
	value: V,
}

// Stores
struct ShareGC<B> where B: Busy {
    queue: LFQueue<B>,
}
```

### References

[PQ Unlocked: Lock-Free Priority Queue](https://tstentz.github.io/418proposal/)
[Skip List Data Structure](https://www.mydistributed.systems/2021/03/skip-list-data-structure.html)
