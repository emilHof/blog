# Race Conditions Abound

After spending some time working on the _thread safe_ implementation of `SkipList`, it is
becoming more and more apparent that getting concurrent `insert` and `remove` to work is
anything but trivial, at least to me. The many pre-conditions and subtle corner cases that need
to be accounted for are posing quite the challenge. There are a few obvious ones, yet solving
those simply bring more varied ones to light. In the following short post I'll try to summarize
a few of the things I have been struggling with.

### Concurrent `remove`

Interestingly enough, `remove` seemed to be simpler to implement than `insert`. This may of
course change in the future, as I discover that some inherent flaw in remove is making it
incompatible with `insert`, yet as for right now it weathers any fuzz test I throw at it.

As of the writing of this post, `removes` implementation looks like this:

```rust
pub fn remove(&self, key: &K) -> Option<(K, V)>
where
	K: Send,
	V: Send,
{
	unsafe {
		match self.find(key, false) {
			SearchResult {
				target: Some((target, mut hazard)),
				mut prev,
				mut level_hazards,
			} => {
				let mut target = target.as_ptr();

				// Set the target state to being removed
				// If this errors, it is already being removed by someone else
				// and thus we exit early.
				if (*target).set_removed().is_err() {
					return None;
				}

				let key = core::ptr::read(&(*target).key);
				let val = core::ptr::read(&(*target).val);
				let mut height = (*target).height();

				while let Err(new_height) = self.unlink(target, height, prev, level_hazards) {
					(target, height, prev, hazard, level_hazards) = {
						if let SearchResult {
							target: Some((target, hazard)),
							prev,
							level_hazards,
						} = self.find(&key, true)
						{
							(target.as_ptr(), new_height, prev, hazard, level_hazards)
						} else {
							break;
						}
					};
				}

				// We will be the only thread to retire this node, as we were the ones
				// who set it to removed
				self.retire_node(target);
				// We see if we can drop some pointers in the list.
				self.garbage.domain.eager_reclaim();
				self.state.len.fetch_sub(1, Ordering::Relaxed);

				Some((key, val))
			}
			_ => None,
		}
	}
}
```
