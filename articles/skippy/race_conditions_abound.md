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

<br></br>

### The Plan

It may be a good idea to start with insertions.

Suppose we have a 4 `Node`s set up as follows:

```rust
[head]---------------------------------->[14];
[head]-------->[4]---------------------->[14];
[head]-->[1]-->[4]--------------->[13]-->[14];
[head]-->[1]-->[4]-->[6]--------->[13]-->[14];
[head]-->[1]-->[4]-->[6]-->[10]-->[13]-->[14];
```

We want to insert a `Node` with a key of `11`. After a search for the insert position, our
`prev` array should look as follows:

```rust
[
	(head, 14),
	(4, 14),
	(4, 13),
	(6, 13),
	(10, 13)
]
```

Now just as we finish our search, `6` gets tagged for removal.

```rust
[head]---------------------------------->[14];
[head]-------->[4]---------------------->[14];
[head]-->[1]-->[4]--------------->[13]-->[14];
[head]-->[1]-->[4]-->[6]--------->[13]-->[14];
[head]-->[1]-->[4]-->[6]-->[10]-->[13]-->[14];
                      r
```

We begin the insertion of `11` by attempting to link it between `10` and `13`.
This succeeds as `10` and `11` are both not tagged for removal and `13` is greater than `11`.
While we do this `6` has been unlinked at level 2 giving us

```rust
[head]----------------------------------------->[14];
[head]-------->[4]----------------------------->[14];
[head]-->[1]-->[4]---------------------->[13]-->[14];
[head]-->[1]-->[4]---------------------->[13]-->[14];
[head]-->[1]-->[4]-->[6]-->[10]-->[11]-->[13]-->[14];
                      r
```

Now we get to level 2 for `Node` `11`. Our `prev` array tells us that we have `6` as the
predecessor of `11` at this level. We see that `6` is removed which means we could potentially
be linking to a dead node. To avoid this we stop building the `Node` here and re-do our search.
From this search we get a `prev` array of:

```rust
[
	(head, 14),
	(4, 14),
	(4, 13),
	(4, 13),
	(10, 11)
]
```

When we now continue to build the next level we pick up at level 2. We check our `prev` at this
level and find `4` to be our predecessor. We do a compare and exchange expecting the old value
to be `13`. After it succeeds our list looks as follows:

```rust
[head]----------------------------------------->[14];
[head]-------->[4]----------------------------->[14];
[head]-->[1]-->[4]---------------------->[13]-->[14];
[head]-->[1]-->[4]--------------->[11]-->[13]-->[14];
[head]-->[1]-->[4]-->[6]-->[10]-->[11]-->[13]-->[14];
                      r
```

As we are building the levels `6` has now been fully removed and another thread has begun to
insert a `Node` with `key` `8` giving us:

```rust
[head]----------------------------------------->[14];
[head]-------->[4]----------------------------->[14];
[head]-->[1]-->[4]-->[8]---------------->[13]-->[14];
[head]-->[1]-->[4]-->[8]--------->[11]-->[13]-->[14];
[head]-->[1]-->[4]-->[8]-->[10]-->[11]-->[13]-->[14];
```

When we now try to build the next level of `11` we try to do a compare and exchange on `4`
expecting the old value to be `13`. Yet, `4` now points at `8` at level 3 and thus this fails.
We must search again, to find what the new state is. While we search `11` gets tagged as being
removed. Since we are now no longer technically part of the list, another thread inserts a new
`11`. The prev arrays of the search for the old, now _removed_ `11` and the new `11` both look
like this:

```rust
[
	(head, 14),
	(4, 14),
	(8, 13),
	(8, 11),
	(10, 11)
]
```

The list looks as follows:

```rust
[head]----------------------------------------->[14];
[head]-------->[4]----------------------------->[14];
[head]-->[1]-->[4]-->[8]---------------->[13]-->[14];
[head]-->[1]-->[4]-->[8]--------->[11]-->[13]-->[14];
[head]-->[1]-->[4]-->[8]-->[10]-->[11]-->[13]-->[14];
                                    r
```

Even though the old `11` is marked as removed, the thread will still try to finish building
all the levels. This may end up being a flaw in the design, yet, that is still tbd for me. The
thread that is inserting the new `11` also succeeds in its first insertion, since the
propositions

```rust
if (!next.is_null() && (*next).key <= (*new_node).key && !(*new_node).removed())
	|| prev.as_ref().removed()
{
	return Err(i)
}
```

evaluates to `false` for both the old `11` as `next` and `10` as `prev`/predecessor. Both
threads succeed at building their next levels.

```rust
[head]------------------------------------------------>[14];
[head]-------->[4]------------------------------------>[14];
[head]-->[1]-->[4]-->[8]---------------->[11]-->[13]-->[14];
[head]-->[1]-->[4]-->[8]---------------->[11]-->[13]-->[14];
[head]-->[1]-->[4]-->[8]-->[10]-->[11]-->[11]-->[13]-->[14];
                                           r
```

It is important to note at this point, that the tread that is actively working on deleting `11`
cannot succeed until `11` is completely finished. This is because of this check:

```rust
if let Err(_) = prev.as_ref().levels[i].as_std().compare_exchange(
	node,
	new_next,
	Ordering::SeqCst,
	Ordering::SeqCst,
) {
	return Err(i + 1);
}
```

The deleting thread expects `prev.levels[i]` to point at `node` being deleted, which will not
be true up to the top level until we completely finished building the `Node`. Since the old
`11` is now at full height, this process will begin.
