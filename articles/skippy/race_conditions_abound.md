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

                    'unlink: while let Err(new_height) = self.unlink(target, height, prev, level_hazards) {
                        (target, height, prev, hazard, level_hazards) = {
                            loop {
                                if let SearchResult {
                                    target: Some((new_target, hazard)),
                                    prev,
                                    level_hazards,
                                } = self.find(&key, true)
                                {
                                    if !core::ptr::eq(new_target.as_ptr(), target) {
                                        continue;
                                    }
                                    break (new_target.as_ptr(), new_height, prev, hazard, level_hazards)
                                } else {
                                    break 'unlink;
                                }
                            }
                        };
                    }
                    // we are the only thread that has permission to drop this node.
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

When we call `remove` we first see if any `Node` with the given `key` exists in our `SkipList`.
If we are not able to find one, we just return `None` as we are done. If we do find a `Node` we
need to try and tag it with `removed`. If we are not able to succeed that means another thread
has started to remove it before we have, so we just exit early with `None`. If we do succeed,
then the next step is to read the `key` and `val` from our `Node` and get the height of its
`levels`. We then keep trying to logically unlink our `Node` from our list, picking up where we
left off with the returned height every time we fail to fully unlink the `Node`. Each time the
unlink method errors we search for our target `Node` again, explicitly allowing removed nodes
to be returned from the search. Note that we only try to unlink the `Node` if the pointer
matches our actual target.

Once we are
completely finished with unlinking we retire our `Node` pointer, and try to deallocate it,
disregarding whether we succeed or fail. Then we decrement the length of our list and exit with
the `key` and `val`.

### Tackling `insert`

The next part of the puzzle is implementing `insert` in a way that is race-free and compatible
to be run concurrently alongside `remove`. This has turned out to be a much more significant
challenge than I expected. As mentioned before, there are a lot of small edge cases that
occur when nodes, potentially of the same value, are inserted alongside one another, and these
conditions only get worse once `remove`s are executed at the same time.

As an example, we could be inserting a new `Node` at some value, which immediately gets tagged
with `removed`. That means there could then be another `Node` of the same value inserted next
to ours since the first `Node` now technically no longer exists in the list. We are still
trying to build the first `Node` though, so on later searches for the insertion point, we need
to be careful that the previous `Node`s we are linking to are updated correctly. One way to
ensure this is by making search always return the first instance of a `Node` with a given key
if we allow removed `Node`s to be returned. We encode this invariant in the following line of
`find`:

```rust
if (*next).key < *key || ( (*next).key == *key && (*next).removed() && !allow_removed ) => {
```

After this check we move our current pointer to the next pointer, basically progressing
forwards on the same level. We only do this if the next key is less then ours, or it is
equal to the current key yet the next node is tagged as removed and we are not allowing returns
of removed `Node`s. So if we are looking for only _lively_ `Node`s we will not stop until
we either find an insertion point that is at a `Node` that has not been removed or at the end
of all of our _removed_ `Node`s with the `prev[0]` being the last _removed_ `Node`.

Here is another edge case. What if we are inserting a `Node`, and we are not done building all
of the levels yet. What if we are moving up the levels and linking to a `Node` that is
concurrently being deleted. That means we are setting `node.levels[i] = next` where next is
being removed. What if we discovered the insertion point where some
`prev_node.levels[i] = next`. What if we switch `prev_node.levels[i] = node` and set
`node.levels[i] = next`. We need to make sure that `next`, which is being removed, does not
think it has successfully unlinked itself, while in actuality `node.levels[i]` is now pointing
at `next` and we could run into a _use-after-free_. To ensure this does not happen we cache
`prev_node.levels[i]` during `find` and then make sure `prev_node.levels[i]` is still the
same when we are doing the compare and exchange later.

```rust
let (prev, next) = previous_nodes[i];

/*
	Some more things here...
*/

if let Err(other) = prev.as_ref().levels[i].as_std().compare_exchange(
	next,
//  ^^^^------------- Ensure that next is still the same.
	new_node,
	Ordering::SeqCst,
	Ordering::SeqCst,
) {
```

All that being said, there obviously needs to be a plan for how to insert and remove `Node`s
as soundly as sound as possible. One of the best ways to do this would seem to be proving
that operations will have deterministic outcomes, no matter the contention.

### The Plan

It may be a good idea to start with insertions.

Suppose we have a 4 `Node`s set up as follows:

```
head------------------->14;
head------>4----------->14;
head-->1-->4----------->14;
head-->1-->4-->6------->14;
head-->1-->4-->6-->10-->14;
```
