# The Racy Case

Consider the following case: We want to remove some `Node` with key `7`. Currently our list
looks as follows:

```rust
[head]-------------->[10]; // level 3
[head]-->[4]-->[7]-->[10]; // level 2
[head]-->[4]-->[7]-->[10]; // level 1
```

When we try to unlink at `level 2` we perform the following checks:

```rust
unsafe fn unlink<'a>(
	&self,
	node: &'a NodeRef<'a, K, V>,
	height: usize,
	previous_nodes: [(NodeRef<'a, K, V>, *mut Node<K, V>); HEIGHT],
) -> Result<(), usize> {
	if self.is_head(node.as_ptr()) {
		panic!()
	}

	for (i, (prev, next)) in previous_nodes.iter().enumerate().take(height).rev() {
		let new_next = node.levels[i].as_std().load(Ordering::Acquire);

		// 1.
		if prev.removed() {
			return Err(i + 1);
		}

		// 2.
		if core::ptr::eq(prev.levels[i].load_ptr(), new_next) {
			continue;
		}

		// 3.
		if !core::ptr::eq(node.as_ptr(), *next) {
			return Err(i + 1);
		}

		// 4.
		if let Err(_) = prev.as_ref().levels[i].as_std().compare_exchange(
			node.as_ptr(),
			new_next,
			Ordering::SeqCst,
			Ordering::SeqCst,
		) {
			return Err(i + 1);
		}
	}

	Ok(())
}
```

Crucially, at step `1.` we try to avoid unlinking from a `removed` `Node`, as this could cause
us to remain linked to some other `Node` given relinking of `removed` occurs before we can
unlink. Now this check at `1.`

```rust
// 1.
if prev.removed() {
	return Err(i + 1);
}
```

should suffice in ensuring this does not happen right? But what if we pass this check and move on
to the `compare_exchange`; yet, before we can actually swap `prev` is suddenly marked as
`removed` and `level 2` is unlinked. We would not be aware of this and still perform the
`compare_exchange` of `node` and `new_next` which would likely succeed.

A very similar issue also arises when linking a new `Node`.

One way to avoid this would be to somehow encode the `removed` tag into our `AtomicPtr` in `prev`
so would `Err(_)` at step `4.` due to `node` now being a different pointer. This could be
done by using the lower, insignificant digits of our `AtomicPtr` as a place to store _tags_.
