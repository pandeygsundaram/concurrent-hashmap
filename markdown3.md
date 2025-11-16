# Concurrent HashMap in Rust - Part 3 Walkthrough

## Table of Contents

1. [Introduction](#introduction)
2. [Session Overview](#session-overview)
3. [Setting Up Basic Functionality](#setting-up-basic-functionality)
   - [Implementing the `new()` Constructor](#implementing-the-new-constructor)
   - [Implementing `with_capacity()`](#implementing-with_capacity)
4. [Writing Basic Tests](#writing-basic-tests)
   - [Test Structure](#test-structure)
   - [Empty Map Tests](#empty-map-tests)
   - [Insert and Get Tests](#insert-and-get-tests)
5. [Fixing Drop Implementation Issues](#fixing-drop-implementation-issues)
   - [Null Bin Handling](#null-bin-handling)
   - [Memory Safety in Drop](#memory-safety-in-drop)
6. [Testing Value Drops](#testing-value-drops)
   - [Creating Drop Tracking Types](#creating-drop-tracking-types)
   - [Reference Counting Approach](#reference-counting-approach)
7. [Concurrent Operations](#concurrent-operations)
   - [Thread Safety with Send and Sync](#thread-safety-with-send-and-sync)
   - [Deadlock Discovery and Fix](#deadlock-discovery-and-fix)
8. [Fixing Resize/Transfer Bugs](#fixing-resizetransfer-bugs)
   - [Bin Movement Issues](#bin-movement-issues)
   - [Run Detection Bug](#run-detection-bug)
9. [Implementing Iterators](#implementing-iterators)
   - [Iterator Design Challenges](#iterator-design-challenges)
   - [Traverser Implementation](#traverser-implementation)
   - [Handling Table Forwarding](#handling-table-forwarding)
   - [Stack-based Iteration](#stack-based-iteration)
10. [Porting Java Tests](#porting-java-tests)
    - [Understanding Java Test Structure](#understanding-java-test-structure)
    - [Adapting Tests to Rust](#adapting-tests-to-rust)
11. [Key Takeaways](#key-takeaways)
12. [Next Steps](#next-steps)

---

## Introduction

This is the third part of a live coding series where we port Java's `ConcurrentHashMap` to Rust. In the previous sessions, we implemented the core data structure and basic insertion/lookup functionality. This session focuses on:

- Adding constructor methods
- Writing comprehensive tests
- Fixing critical bugs discovered through testing
- Implementing iterators
- Porting Java's test suite

The speaker is a PhD student at MIT who conducts these streams to build real, production-quality Rust code while exploring advanced concurrency concepts.

---

## Session Overview

**Duration**: ~6 hours  
**Main Goals**:
1. Implement basic API methods (`new()`, `with_capacity()`)
2. Write initial test suite
3. Debug issues found through testing
4. Implement concurrent iteration
5. Port Java test cases

**Key Learning Points**:
- Testing concurrent data structures
- Memory safety in concurrent contexts
- Iterator design patterns
- Debugging race conditions and deadlocks

---

## Setting Up Basic Functionality

### Implementing the `new()` Constructor

The session starts by realizing the HashMap doesn't have a way to be constructed! Looking at the Java implementation:

```java
public ConcurrentHashMap() {
    // Empty constructor - fields use default values
}
```

In Java, the constructor is empty because fields are initialized lazily when first accessed. The Rust equivalent:

```rust
impl<K, V> HashMap<K, V, RandomState> {
    pub fn new() -> Self {
        Self {
            table: Atomic::null(),
            count: AtomicUsize::new(0),
            transfer_index: 0,
            size_ctl: 0,
            build_hasher: RandomState::new(),
        }
    }
}
```

**Key Insight**: The `new()` method is only available when `S = RandomState` (the default hasher). For custom hashers, use `with_hasher()`.

### Implementing `with_capacity()`

Looking at Java's capacity-based constructor:

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity));
    this.sizeCtl = cap;
}
```

The Rust implementation:

```rust
pub fn with_capacity(n: usize) -> Self {
    assert_ne!(n, 0, "capacity must be non-zero");
    
    let size = n.next_power_of_two();
    let size = if size > MAXIMUM_CAPACITY {
        MAXIMUM_CAPACITY
    } else {
        size
    };
    
    let mut map = Self::new();
    map.size_ctl = size as isize;
    map
}
```

**Important Notes**:
- Uses `next_power_of_two()` instead of Java's `tableSizeFor()` - they do the same thing
- Panics on zero capacity (could return `Result` instead, but decided against it)
- `size_ctl` stores the initial capacity until first insert

---

## Writing Basic Tests

### Test Structure

Created `tests/basic.rs` for integration tests:

```rust
use flurry::*;

#[test]
fn new() {
    let map: HashMap<usize, usize> = HashMap::new();
    // Should not panic
}
```

**Why Integration Tests?**
- Unit tests are for internal-only functionality
- Integration tests verify public API
- Separate files make organization cleaner

### Empty Map Tests

First test immediately revealed a bug:

```rust
#[test]
fn new() {
    let map: HashMap<usize, usize> = HashMap::new();
    // Panicked in Drop!
}
```

**The Bug**: The `Drop` implementation tried to deallocate a table that was never allocated.

**The Fix**:
```rust
impl<K, V, S> Drop for HashMap<K, V, S> {
    fn drop(&mut self) {
        // Check if table was ever allocated
        if self.table.load(Ordering::SeqCst, &epoch::pin()).is_null() {
            return; // Table was never used
        }
        
        // ... rest of drop logic
    }
}
```

### Insert and Get Tests

```rust
#[test]
fn insert_and_get() {
    let map = HashMap::new();
    let guard = epoch::pin();
    
    map.insert(42, 0);
    
    let v = map.get(&42, &guard).unwrap();
    unsafe {
        assert_eq!(*v, 0);
    }
}
```

**Issues Discovered**:
1. Null bins in allocated tables caused panics
2. Need to check `bin.is_null()` before dereferencing
3. Guard API is awkward for users (needs improvement later)

---

## Fixing Drop Implementation Issues

### Null Bin Handling

When dropping a table, not all bins may be populated:

```rust
for bin in &table.bins {
    let bin = bin.load(Ordering::SeqCst, &guard);
    
    if bin.is_null() {
        continue; // Bin was never used
    }
    
    // Walk the linked list and drop nodes
    // ...
}
```

### Memory Safety in Drop

The drop implementation needs to:
1. Walk all bins
2. For each bin, walk the linked list
3. Drop all nodes
4. Avoid double-frees

```rust
unsafe {
    let node = bin.deref();
    let mut p = node.next.load(Ordering::SeqCst, &guard);
    
    while !p.is_null() {
        let node = p.into_owned();
        p = node.next.load(Ordering::SeqCst, &guard);
        // node dropped here
    }
}
```

---

## Testing Value Drops

### Creating Drop Tracking Types

To verify values are properly dropped:

```rust
#[derive(Clone)]
struct NotifyOnDrop {
    dropped: Arc<AtomicBool>,
}

impl Drop for NotifyOnDrop {
    fn drop(&mut self) {
        self.dropped.store(true, Ordering::SeqCst);
    }
}
```

**Problem**: Can't easily test deferred drops due to epoch-based GC.

### Reference Counting Approach

Better solution using `Arc`:

```rust
#[test]
fn current_kv_dropped() {
    let map = HashMap::new();
    
    let dropped1 = Arc::new(0);
    let dropped2 = Arc::new(0);
    
    // Insert first value
    map.insert(dropped1.clone(), 0);
    assert_eq!(Arc::strong_count(&dropped1), 2); // us + map
    
    // Replace with second value
    map.insert(dropped2.clone(), 0);
    assert_eq!(Arc::strong_count(&dropped1), 2); // Still 2 (deferred GC)
    assert_eq!(Arc::strong_count(&dropped2), 2);
    
    // Drop map
    drop(map);
    
    // Now both should be dropped
    assert_eq!(Arc::strong_count(&dropped1), 1);
    assert_eq!(Arc::strong_count(&dropped2), 1);
}
```

**Key Learning**: Can't control when epoch-based GC runs, but can verify drops happen when map is dropped.

---

## Concurrent Operations

### Thread Safety with Send and Sync

Attempting concurrent inserts:

```rust
#[test]
fn concurrent_insert() {
    let map = Arc::new(HashMap::new());
    let map1 = map.clone();
    let map2 = map.clone();
    
    let t1 = thread::spawn(move || {
        for i in 0..64 {
            map1.insert(i, 0);
        }
    });
    
    // Error: HashMap is not Send!
}
```

**The Problem**: Raw pointers are not `Send` or `Sync` by default.

**The Solution**: Implement `Send` and `Sync` manually:

```rust
unsafe impl<K, V, S> Send for HashMap<K, V, S>
where
    K: Send,
    V: Send,
    S: Send,
{}

unsafe impl<K, V, S> Sync for HashMap<K, V, S>
where
    K: Sync,
    V: Sync,
    S: Sync,
{}
```

**Safety Argument**: The entire implementation is designed for concurrent access. The raw pointers are managed safely through atomic operations and the epoch-based GC.

### Deadlock Discovery and Fix

Running concurrent inserts **hung indefinitely**:

```
t1 at i: 11
t2 at i: 11
[hangs]
```

**Investigation with GDB**:
```
Thread 2: waiting on mutex at line 298
Thread 3: waiting on mutex at line 662 (in transfer)
```

**Root Cause**: Thread holds bin lock while calling `add_count()`, which may trigger `transfer()`, which tries to acquire the same lock!

```rust
// In put_val():
let _lock = bin.lock(); // Acquire lock

// ... do insert ...

self.add_count(1, ...); // May trigger transfer!
// Still holding lock here!
```

**The Fix**: Drop the lock guard before calling `add_count()`:

```rust
{
    let _lock = bin.lock();
    // ... do insert ...
} // Lock dropped here

self.add_count(1, ...); // Safe now
```

**Key Insight**: Java's `synchronized` blocks have explicit end points. Rust's RAII lock guards are better, but you need to be aware of their scope.

---

## Fixing Resize/Transfer Bugs

### Bin Movement Issues

After fixing the deadlock, gets started failing:

```rust
#[test]
fn concurrent_insert() {
    // ... insert 0..64 from two threads ...
    
    for i in 0..64 {
        let v = map.get(&i, &guard);
        assert!(v.is_some()); // FAILED at random keys!
    }
}
```

**Debugging**: Added extensive print statements to track bin movements:

```
Moving table bin 25: key 16 to high
Moving table bin 27: key 6 to high, key 7 to high
```

But key 7 should go to **low**, not high!

### Run Detection Bug

The problem was in the "run detection" logic during bin movement:

```rust
// Original (WRONG):
while let Some(p) = p_next {
    if p.next.is_some() {  // Check if has next
        let run_bit = p.hash & n;
        if run_bit != last_run_bit {
            // Found different run
        }
        p = p.next;
    } else {
        break;  // WRONG: breaks without checking last node!
    }
}
```

**The Fix**: Check the run bit **before** checking if there's a next node:

```rust
while let Some(p) = p_next {
    let run_bit = p.hash & n;
    if run_bit != last_run_bit {
        // Found different run
        last_run = p;
        last_run_bit = run_bit;
    }
    
    if p.next.is_none() {
        break;
    }
    p = p.next;
}
```

**Lesson**: The order of checks matters! The original Java code had them in the right order, but during porting they got moved around.

---

## Implementing Iterators

### Iterator Design Challenges

Iterating over a concurrent HashMap is tricky:
- Table can resize during iteration
- Need to handle forwarding nodes
- Must not visit same key twice
- Can't hold locks for entire iteration

**Java's Approach**: Use a "traverser" that:
1. Iterates bins in order
2. When it hits a forwarding node, descends into the new table
3. Uses a stack to track where to return to
4. Ensures each key visited exactly once

### Traverser Implementation

Created `src/iter/traverser.rs`:

```rust
pub struct BinEntryIter<'g, K, V> {
    table: Option<&'g Table<K, V>>,
    prev: Option<&'g Node<K, V>>,
    index: usize,
    base_index: usize,
    base_limit: usize,
    base_size: usize,
    stack: Option<Box<TableStack<'g, K, V>>>,
    spare: Option<Box<TableStack<'g, K, V>>>,
}
```

**Key Fields**:
- `table`: Current table being iterated
- `prev`: Last node yielded (for walking linked lists)
- `index`: Current bin index
- `base_*`: Track original table dimensions
- `stack`: Stack of tables when following forwarding nodes
- `spare`: Recycled stack nodes (avoid allocations)

### Handling Table Forwarding

When encountering a forwarding node:

```rust
match bin {
    BinEntry::Moved(next_table) => {
        // Save current state
        self.push_state(table, index, n);
        
        // Descend into forwarded table
        self.table = Some(next_table);
        self.prev = None;
        
        continue; // Process same bin in new table
    }
    BinEntry::Node(node) => {
        // Normal iteration
    }
}
```

### Stack-based Iteration

The most complex part: managing the stack of tables.

**Visual Example**:

```
Original table (16 bins):
[0][1][2][3]...[15]
     ↓ forwarded
     
Table 1 (32 bins):
[0][1][2]...[16][17]...[31]
          ↓ forwarded
          
Table 2 (64 bins):
[0][1][2]...[32][33]...[63]
```

To iterate bin 1:
1. Visit bin 1 in original (forwarded → go deeper)
2. Visit bin 1 in table 1 (forwarded → go deeper)
3. Visit bin 1 in table 2 (has nodes)
4. Visit bin 17 in table 2 (high half of bin 1 from table 1)
5. Pop back to table 1
6. Visit bin 17 in table 1 (forwarded → go deeper)
7. Visit bin 17 in table 2
8. Visit bin 49 in table 2 (high half)
9. Pop back to original
10. Move to bin 2

**Implementation**:

```rust
fn recover_state(&mut self, n: usize) {
    while let Some(mut s) = self.stack.take() {
        self.index += s.length;
        
        if self.index < n {
            // Haven't checked high half yet - don't pop
            self.stack = Some(s);
            return;
        }
        
        // Pop this level
        n = s.length;
        self.table = s.table;
        
        // Recycle stack node
        s.next = self.spare.take();
        self.spare = Some(s);
    }
    
    // At top level - move to next base bin
    self.index += self.base_size;
    if self.index >= n {
        self.base_index += 1;
        self.index = self.base_index;
    }
}
```

**Key Insight**: This isn't really "recovering state" - it's computing the next bin index across multiple table levels.

---

## Porting Java Tests

### Understanding Java Test Structure

Java's test file (`ConcurrentHashMapCheck.java`) has unusual structure:

```java
static void test(Map s, Object[] key, int size) {
    // Generic test runner
}

static void t1(Map m, Object[] k, int n) {
    // Specific test: get present keys
}

static void t3(Map m, Object[] k, int n) {
    // Specific test: put keys
}

public static void main(String[] args) {
    // Run tests with shuffled keys
}
```

### Adapting Tests to Rust

Created cleaner Rust version:

```rust
const SIZE: usize = 50_000;

#[test]
fn get_present() {
    let map = HashMap::new();
    let keys: Vec<usize> = (0..SIZE).collect();
    
    // First populate
    put_all(&map, &keys);
    
    // Then verify all present
    let guard = epoch::pin();
    let mut found = 0;
    
    for _ in 0..4 {  // Multiple iterations
        for key in &keys {
            if map.get(key, &guard).is_some() {
                found += 1;
            }
        }
    }
    
    assert_eq!(found, SIZE * 4);
}

fn put_all<K, V>(map: &HashMap<K, V>, keys: &[K]) 
where
    K: Hash + Eq + Copy,
    V: Default,
{
    for key in keys {
        map.insert(*key, V::default());
    }
}
```

**Improvements over Java**:
- Clearer function names
- Type safety (no `Object[]`)
- Removed unnecessary parameters
- Better organization

**Results**: Tests passed! The implementation correctly handles:
- 50,000 insertions
- Multiple table resizes
- Concurrent access patterns
- Iterator correctness

---

## Key Takeaways

### 1. Testing Concurrent Data Structures is Hard

- Epoch-based GC makes drop timing non-deterministic
- Need to test both sequential and concurrent access
- Race conditions only appear under specific timing
- Print debugging is still valuable

### 2. Memory Safety Requires Careful Reasoning

Every `unsafe` block needs documentation:

```rust
unsafe {
    // SAFETY: bin was read under `guard` and `guard` is still held.
    // The map guarantees bins are not freed until after all guards
    // that could reference them are dropped.
    bin.deref()
}
```

### 3. Lock Scoping is Critical

Rust's RAII lock guards are great, but you must be aware of their scope:

```rust
// BAD: Lock held too long
let _lock = bin.lock();
do_work();
might_reacquire_lock(); // DEADLOCK!

// GOOD: Explicit scope
{
    let _lock = bin.lock();
    do_work();
} // Lock released
might_reacquire_lock(); // OK
```

### 4. Iterator Design Patterns

For concurrent collections:
- Store guard in iterator (ties lifetime)
- Use references instead of `Shared` internally
- Stack-based traversal for complex structures
- Recycle allocations when possible

### 5. Port Faithfully, Then Improve

1. First port: Match original logic closely
2. Add safety documentation
3. Add tests
4. Refactor for better Rust idioms
5. Optimize

---

## Next Steps

### Immediate TODOs

1. **Implement `remove()`**: The remove operation is missing but should be straightforward following the pattern of `insert()`

2. **Improve Public API**:
   - Hide `Shared` type from public interface
   - Make guard handling more ergonomic
   - Consider builder pattern for configuration

3. **Port Remaining Tests**:
   - Iterator tests
   - Remove tests
   - Edge case tests
   - Stress tests

4. **Set Up CI**:
   - GitHub Actions for testing
   - Miri for unsafe code checking
   - Clippy for lints
   - Benchmarks

### Future Enhancements

1. **Optimizations**:
   - Counter cells (sharded counting)
   - Tree bins for long chains
   - Custom load factor
   - Resize hints

2. **API Improvements**:
   - Entry API (like std HashMap)
   - Bulk operations
   - Parallel iterators (via Rayon)
   - Custom hashers

3. **Documentation**:
   - Architecture overview
   - Safety guarantees
   - Performance characteristics
   - Migration guide from std::HashMap

### Potential Issues to Investigate

1. **Memory Ordering**: Some atomics use `SeqCst` which could be relaxed
2. **Garbage Collection**: Epoch-based GC can delay drops - is this acceptable?
3. **Performance**: How does it compare to other concurrent maps?
4. **API Ergonomics**: The guard requirement is awkward

---

## Conclusion

This session demonstrated:
- How to test concurrent data structures
- Debugging techniques for race conditions
- Iterator implementation patterns
- Importance of faithful porting before optimization

The concurrent HashMap implementation is now:
- ✅ Constructible
- ✅ Supports insert and get
- ✅ Thread-safe
- ✅ Handles resizing correctly
- ✅ Iterable
- ✅ Passes basic Java test suite

Next session (in a few months) will likely focus on a different project, as the core porting work is largely complete. The community is encouraged to:
- Port remaining tests
- Implement missing features
- Improve documentation
- Run benchmarks
- Submit PRs

**Repository**: [github.com/jonhoo/flurry](https://github.com/jonhoo/flurry)

---

## Appendix: Key Code Snippets

### Constructor Implementation

```rust
impl<K, V> HashMap<K, V, RandomState> {
    pub fn new() -> Self {
        Self::with_hasher(RandomState::new())
    }
    
    pub fn with_capacity(n: usize) -> Self {
        assert_ne!(n, 0);
        let size = n.next_power_of_two().min(MAXIMUM_CAPACITY);
        
        let mut map = Self::new();
        map.size_ctl = size as isize;
        map
    }
}

impl<K, V, S> HashMap<K, V, S> {
    pub fn with_hasher(hasher: S) -> Self {
        Self {
            table: Atomic::null(),
            count: AtomicUsize::new(0),
            transfer_index: 0,
            size_ctl: 0,
            build_hasher: hasher,
        }
    }
}
```

### Drop Implementation

```rust
impl<K, V, S> Drop for HashMap<K, V, S> {
    fn drop(&mut self) {
        let guard = epoch::pin();
        let table = self.table.load(Ordering::SeqCst, &guard);
        
        if table.is_null() {
            return; // Never allocated
        }
        
        unsafe {
            let table = table.deref();
            for bin in &table.bins {
                let bin = bin.load(Ordering::SeqCst, &guard);
                if bin.is_null() {
                    continue; // Empty bin
                }
                
                // Walk and drop all nodes in this bin
                // ...
            }
        }
    }
}
```

### Iterator Next Implementation

```rust
impl<'g, K, V> Iterator for BinEntryIter<'g, K, V> {
    type Item = &'g Node<K, V>;
    
    fn next(&mut self, guard: &'g Guard) -> Option<Self::Item> {
        loop {
            // Try to walk current bin
            if let Some(prev) = self.prev {
                let next = prev.next.load(Ordering::SeqCst, guard);
                if !next.is_null() {
                    self.prev = Some(unsafe { next.deref() });
                    return self.prev;
                }
            }
            
            // Need to move to next bin
            if self.base_index >= self.base_limit 
                || self.table.is_none() 
                || self.index >= n 
            {
                self.prev = None;
                return None;
            }
            
            // Load next bin
            let bin = self.table.unwrap().bins[self.index]
                .load(Ordering::SeqCst, guard);
            
            match bin {
                BinEntry::Moved(next_table) => {
                    // Follow forwarding pointer
                    self.push_state(self.table, self.index, n);
                    self.table = Some(next_table);
                    self.prev = None;
                }
                BinEntry::Node(node) => {
                    if let Some(stack) = self.stack {
                        self.recover_state(n);
                    } else {
                        self.index += self.base_size;
                        if self.index >= n {
                            self.base_index += 1;
                            self.index = self.base_index;
                        }
                    }
                    self.prev = Some(node);
                    return self.prev;
                }
            }
        }
    }
}
```

---

**Session Duration**: ~6 hours  
**Lines of Code Added**: ~800  
**Bugs Fixed**: 4 major (drop, deadlock, transfer, iterator)  
**Tests Added**: 10+  
**Coffee Consumed**: Not disclosed 😄