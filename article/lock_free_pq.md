# Road to Lockfree Heap

### What is Lockfree

Among the different concurrency enabling paradigms, lockfree tries to be unique by offering a
way to operate on shared memroy without the need for locks. In scenarious where lots of cpus
are contending on inserts and deletions, acquiring locks can impose a significant increase
in runtime.

The idea of using a lockfree data structure to avoid the pitfalls of locks, is definitely not a
new one. Thanks to all the insanely smart scientists and engineers that have worked on this problem
we have a multitude of different implementations of mechanisms that enable lockfreedom.

It should also definitely be mentioned that it can often be debatable how much better lockfree
data structures really are. Much of this uncertainty could be due to how difficult it is to impliment
lockfree code correctly and efficiently.

### Why a Binary Heap

### Challenges

### Initial Plan

```rust
pub struct LFHeap<K, V> {
    /* some data */
}

struct HeapNode<K,V> {
    key: K,
	value: V
}
```

### References

[PQ Unlocked: Lock-Free Priority Queue](https://tstentz.github.io/418proposal/)
