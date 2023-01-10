# Probing for Edge Cases and Fuzzing

### Current State

While `skippy-rs` is now in `alpha`, it should probably be categorized as a `pre-alpha` crate.
It is now at the point where many of the race conditions that still exist in the code are
becoming harder and harder to find, simply due to their increased rarity.

In full honesty, much of my work on this crate so far has been guided by the prior efforts
of the `crossbeam` contributor team. `skippy-rs` is extremely similar to `crossbeam-skiplist`
with the main difference being the garbage collector. As stated in previous posts, `skippy-rs`
uses **Hazard Pointers** for its garbage collection where as `crossbeam-skiplist` uses
an **Epoch** based collection scheme.

Another, as of now, distinctive feature of `skippy-rs` is the ability to have a _non-concurrent_
`SkipList`,which is much faster, and convert that `SkipList` into a _concurrent_ one, on the fly.

### Fuzzing
