# Road to Lock Free PQ

### What does it mean to be "Lock Free"

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

### Some Skip List Details

- Each level should have half the Nodes of the next lower level.
- This gives us a tree-like structure where the number of Nodes in a level is $2^{h-l}$, where $h$ is
  height of the tree and $l$ is the level.
- We want to maintain a somewhat _clean_ structure where nodes are roughly evenly distributed across a level.
- To _insert_ a Node, we find the place to insert it, and start building a tower. To do so we flip a coin
  with a $50/50$ probability distribution. We keep adding nodes to our tower until we get, say tails. Then we stop

### Initial Plan

```rust
pub struct SkipList {
    head: ListNode<K, V>,  // Our head node
	height: u8,  // The max height is < 64, so u8 should suffice
	length: usize,  // Should be < usize::<MAX
}

struct ListNode<K, V> {
    key: K,
	value: V,

	left: *mut ListNode,
	right: *mut ListNode,
	top: *mut ListNode,
	bottom: *mut ListNode,
}
```

### References

[PQ Unlocked: Lock-Free Priority Queue](https://tstentz.github.io/418proposal/)
[Skip List Data Structure](https://www.mydistributed.systems/2021/03/skip-list-data-structure.html)
