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

### MaybeTagged

By storing additional `tag` information in the lower bits of our `AtmoicPtr` any thread with outdated
information would fail when `compare_exchange`ing the `untagged` pointer. Of course, messing with the
bits of a pointer can cause horrible UB, so this requires some amount of care to implement. In comes,
`MaybeTagged`:

```rust
pub(crate) struct MaybeTagged<T>(AtomicPtr<T>);
```

In essence, `MaybeTagged` stores `*mut T` in its inner `AtomicPtr<T>` which has potentially had
bits set in its lower, insignificant bit range. Given a tag of `usize` and a pointer, it
composes the two and stores the tagged `*mut` inside of the `AtomicPtr`.

```rust
pub(crate) fn store_composed(&self, ptr: *mut T, tag: usize) {
	let tagged = Self::compose_raw(ptr, tag);

	unsafe {
		self.0
			.as_std()
			.store(tagged, std::sync::atomic::Ordering::Release);
	}
}

#[inline]
fn compose_raw(ptr: *mut T, tag: usize) -> *mut T {
	usize_to_ptr_with_provenance(
		(ptr as usize & !unused_bits::<T>()) | (tag & unused_bits::<T>()),
		ptr,
	)
}
```

On loads, it ensures that a `*mut` is never returned with its lower bits still set, as to not
cause SEGFAULTs and UB.

```rust
pub(crate) fn load_ptr(&self) -> *mut T {
	self.load_decomposed().0
}

pub(crate) fn load_decomposed(&self) -> (*mut T, usize) {
	let raw = unsafe { self.0.as_std().load(std::sync::atomic::Ordering::Acquire) };
	Self::decompose_raw(raw)
}

#[inline]
fn decompose_raw(raw: *mut T) -> (*mut T, usize) {
	(
		usize_to_ptr_with_provenance(raw as usize & !unused_bits::<T>(), raw),
		raw as usize & unused_bits::<T>(),
	)
}
```

Now, with this `MaybeTagged` scheme in place, we can pass around our protected `*mut Node`s
inside of a `NodeRef`, which holds the `*mut Node` along with the `HazardPointer` that ensures
it's protection.

```rust
#[allow(dead_code)]
struct NodeRef<'a, K, V> {
    node: NonNull<Node<K, V>>,
    _hazard: HazardPointer<'a>
}

impl<'a, K, V> NodeRef<'a, K, V> {
    pub(crate) fn from_maybe_tagged(maybe_tagged: &MaybeTagged<Node<K, V>>) -> Option<Self> {
        let mut _hazard = HazardPointer::new();
        let mut ptr = maybe_tagged.load_ptr();

        _hazard.protect_raw(ptr);

        let mut v_ptr = maybe_tagged.load_ptr();

        while !core::ptr::eq(ptr, v_ptr) {
            ptr = v_ptr;
            _hazard.protect_raw(ptr);

            v_ptr = maybe_tagged.load_ptr();
        }

        if ptr.is_null() {
            None
        } else {
            unsafe {
                Some(NodeRef {
                    node: core::ptr::NonNull::new_unchecked(ptr),
                    _hazard,
                })
            }
        }
    }
}
```
