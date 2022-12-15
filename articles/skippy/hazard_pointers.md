# Hazard Pointers Use-Case

### Question:

How do we deallocate Nodes safely to ensure no **use-after-free** occurs?

### Answer:

We use Hazard Pointers to ensure memory is only deallocated once no thread is referring to it.

### The Solution

Hazard Pointers provide a safe way to recalim memory in a concurrent setting. In the case of lock
free algorithms and data structures, they allow for shared memory access without risking reads
of deallocated memory.

This safety is eached through a sort of _garbage collection_ where threads indicate that they
are currently accessing a pointer, by placing it into a **Hazard Pointer**. The collection of
all Hazard Pointers is surveyed by a _Collector_ or a _Domain_ that prevents the deallocation
of any pointed to memory location as long as at least on of the threads is storing the pointer
in its Hazard Pointer. This is, of course, an very oversimplified explaination, yet it works
should suffice for now.

Luckily much smarter programmers have already worked out an implementation of Hazard Pointers in
Rust. Namely the [haphazard](https://github.com/jonhoo/haphazard) crate, which is being
maintained by [John Gjengset](https://github.com/jonhoo).

### Usage

John's crate provides two key types we will be using. The `HazardPointer` itself and the `Domain`
which will act a bit like a garbage collector.

A basic usage of the Hazard Pointer could look like this:

```rust
// We create our domain and we have to ensure that every thread that interacts with our data
// also uses this domain.
let domain = Domain::new(&());

// We fetch the data we want to access.
let node_ptr: AtomicPtr<Node<_,_>> = /* ... get node pointer from somewhere */;
let mut h_ptr = HazardPointer::new_in_domain(&domain);  // Register HP with our domain.

// Now we protect our node pointer with our HazardPointer
let protected_node  = unsafe { x.load(&mut h_ptr).expect("ptr to be valid") };

/*
Do things with protected_node here...
*/

// We let go of our reference to the node
h_ptr.reset_protection()

// Any use of protected_node now is invalid!
// let _ = protected_node.val;
```

To safely drop a value within this framework we use the `.retire_in()` on a pointer we want to
let go of, and run `.eager_reclaim()` on our `Domain`. We need to be careful though, so that
there will never be two threads retiring the same pointer, as this will cause UB.

```rust
let node_ptr: AtomicPtr<Node<_,_>> = /* ... get the pointer we want to drop */;

unsafe { node_ptr.retire_in(&domain) };

// Once no Hazard Pointers are protecting the pointer it will be removed next time we run:
let reclaimed: usize = domain.eager_reclaim();

assert!(reclaimed, 1);
```

### Using it Within Skippy

```rust
pub struct SkipList<K, V> {
    head: NonNull<Head<K, V>>,
    state: ListState,
	// add a garbage collector
	garbage: Can,
}

impl<K, V> Clone for SkipList<K, V> {
	fn clone(&self) -> Self{
		SkipList {
			head: self.head.clone(),
			state: ListState.clone(),
			// add a garbage collector
			garbage: Can.clone(),
		}
	}
}

struct Can {
	domain: &haphazard::Domain,
	pointer: haphazard::HazardPointer,
}

impl Clone for Can {
	fn clone(&self) -> Self{
		Can {
			domain: self.domain,
			pointer: HazardPointer::new_in_domain(&self.domain()),
		}
	}
}
```

Might need a special method on `Domain`, namely `.retire_ptr_with()`, which could look something
like this:

```rust
/// Retire a `ptr` with a custom deleter function.
/// `T` must be `Send` since it may be reclaimed by a different thread.
///
/// # Safety
/// 1. No [`HazardPointer`] will guard `ptr` from this point forward.
/// 2. `ptr` has not already been retired unless it has been reclaimed since then.
/// 3. `ptr` is valid as `&T` until `self` is dropped.
/// 4. `f` must deallocate T, else memory will be leaked.
impl<F> Domain<F> {
pub unsafe fn retire_ptr_with<T, D>(&self, ptr: *mut T, deleter: unsafe fn(*mut dyn Reclaim)) -> usize
where
T: Send,
{
	let retired = Box::new(unsafe {
		Retired::new(self, ptr, deleter)
	});

	self.push_list(retired)
}
}
```

Another option may be implementing `Pointer<T>` for `Node<K, V>` in the following way:

```rust
Deallocator<T>(T);

impl<K,V> Drop for Deallocator<Node<K,V>> {
	fn drop(&mut self) {
		Node::drop(self.0);
	}
}

unsafe impl<K, V> Pointer<Node<K, V>> for Deallocator<Node<K, V>> {
	fn into_raw(self) -> *mut T {
	    self.0
	}

	unsafe fn from_raw(ptr: *mut T) -> Self {
		Deallocator::from_raw(ptr)
	}
}
```

The second option adds additional indirection, which may not be entirely necessary if
`retire_ptr_with()` exists as an option. Yet, this option has the significant advantage of not
requiring the maintainers of `haphazard` to go out of their way to accomodate me!
