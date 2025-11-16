# Porting Java's ConcurrentHashMap to Rust: A Complete Guide

**Based on Jon Gjengset's Live Coding Stream**

---

## Table of Contents

1. [Introduction & Overview](#1-introduction--overview)
2. [Core Concepts](#2-core-concepts)
3. [Architecture & Design](#3-architecture--design)
4. [Memory Management](#4-memory-management)
5. [Core Operations](#5-core-operations)
6. [Locking Strategy](#6-locking-strategy)
7. [The Resizing Process](#7-the-resizing-process)
8. [Implementation Details](#8-implementation-details)
9. [Java vs Rust Differences](#9-java-vs-rust-differences)
10. [Current Status & Future Work](#10-current-status--future-work)
11. [Testing Strategy](#11-testing-strategy)
12. [Key Takeaways](#12-key-takeaways)
13. [Appendices](#13-appendices)

---

# 1. Introduction & Overview

## What is ConcurrentHashMap?

A `ConcurrentHashMap` is a high-performance hash table that allows **multiple threads** to read and write simultaneously without using a single global lock.

### The Problem with Standard Approaches

**Naive Approach:**
```rust
// Just wrap HashMap in a Mutex
let map = Arc::new(Mutex::new(HashMap::new()));
```

**Why This Is Slow:**
- Only ONE thread can access the map at a time
- Readers block writers and vice versa
- All cores wait in line for the lock
- Becomes a major bottleneck in multi-threaded applications

## Key Challenges

### 1. Memory Reclamation
**Problem:** How do we know when to free a node?

```
Thread 1: Reading node A
Thread 2: Deletes node A
         → When can we free it?
Thread 1: Still using node A!
```

**Solution:** Epoch-based reclamation (crossbeam-epoch)

### 2. Lock-Free Reads
Readers must never wait for writers, but must always see consistent data.

### 3. Concurrent Resizing
The map must grow when too full. During resizing:
- Readers continue accessing old table
- Writers can help with the resize
- Must maintain correctness throughout

---

# 2. Core Concepts

## Hash Tables and Bins

A hash table maps keys to values using a hash function:

```
Key → Hash Function → Index → Value
"foo" → hash("foo") → 42 → "bar"
```

### Structure

```
Table:
┌─────┬─────┬─────┬─────┬─────┐
│ Bin │ Bin │ Bin │ Bin │ Bin │
└──┬──┴─────┴──┬──┴─────┴─────┘
   │           │
   ▼           ▼
 Node        Node → Node → Node
 (key,val)   (linked list)
```

## Memory Ordering and Atomics

### Memory Orderings in Rust

**Relaxed:**
```rust
atomic.store(5, Ordering::Relaxed);
// No synchronization guarantees
```

**Acquire/Release:**
```rust
// Thread 1
atomic.store(5, Ordering::Release);

// Thread 2
let val = atomic.load(Ordering::Acquire);
// Sees all writes before the Release
```

**SeqCst (Sequential Consistency):**
```rust
atomic.store(5, Ordering::SeqCst);
// Strongest guarantee
// All threads see same order
```

## The Happens-Before Relationship

**Definition:** If operation A happens-before operation B, then B sees all effects of A.

---

# 3. Architecture & Design

## FlurryHashMap Structure

```rust
pub struct FlurryHashMap<K, V, S = RandomState> {
    /// Current table (atomic pointer)
    table: Atomic<Table<K, V>>,
    
    /// Number of elements in the map
    count: AtomicUsize,
    
    /// Control field for resizing
    size_ctl: AtomicIsize,
    
    /// Next index to transfer during resize
    transfer_index: AtomicIsize,
    
    /// The new table during resize
    next_table: Atomic<Table<K, V>>,
    
    /// Hash builder
    build_hasher: S,
}
```

## Node Types (BinEntry Enum)

```rust
enum BinEntry<K, V> {
    /// Regular node containing data
    Node(Node<K, V>),
    
    /// Forwarding pointer during resize
    Moved {
        next_table: *const Table<K, V>,
    },
}
```

### Node Structure

```rust
struct Node<K, V> {
    hash: u64,
    key: K,
    value: Atomic<V>,
    next: Atomic<BinEntry<K, V>>,
    lock: Mutex<()>,
}
```

---

# 4. Memory Management

## The Garbage Collection Problem

```rust
// Thread 1: Reading
let node = bin.load(guard);
let value = node.value;  // Using node

// Thread 2: Writing
let old = node.value.swap(new_value);
// Should we drop `old` now?
// NO! Thread 1 might still be using it!
```

## Crossbeam Epoch-Based Reclamation

### Core Idea

**Global Epochs:**
```
Epoch 0 → Epoch 1 → Epoch 2 → Epoch 3 ...
```

**Track When Threads Are Active:**
```
Epoch 1: Thread A pins
         Thread B deletes X
         → X goes into bag[1]

Epoch 2: Thread A unpins

Epoch 3: → Can now free bag[1]
         → All threads have moved past epoch 1
```

## Guards: Tracking Thread Access

```rust
let guard = crossbeam_epoch::pin();
// Thread is now "pinned" to current epoch

let node = atomic.load(Ordering::Acquire, &guard);
// Pointers loaded under guard are protected

// Use node safely...

drop(guard);  
// Can now reclaim memory
```

## Owned vs Shared vs Atomic

### Owned<T>
**Meaning:** You uniquely own this allocation

```rust
let node = Owned::new(Node { ... });
```

### Shared<'g, T>
**Meaning:** A pointer valid during guard lifetime

```rust
let shared = atomic.load(Ordering::Acquire, guard);
```

### Atomic<T>
**Meaning:** An atomic pointer that can be CAS'd

```rust
atomic.store(node, Ordering::Release);
```

---

# 5. Core Operations

## The Get Operation (Reading)

```rust
pub fn get<'g>(&self, key: &K, guard: &'g Guard) 
    -> Option<Shared<'g, V>>
{
    let hash = self.hash(key);
    
    let table = self.table.load(Ordering::SeqCst, guard);
    if table.is_null() {
        return None;
    }
    
    let table = unsafe { table.deref() };
    let bin_i = table.bin(hash);
    let bin = table.bin(bin_i, guard);
    
    if bin.is_null() {
        return None;
    }
    
    let node = unsafe { bin.deref() }.find(hash, key, guard);
    
    if node.is_null() {
        return None;
    }
    
    let node = unsafe { node.deref() };
    Some(node.value.load(Ordering::SeqCst, guard))
}
```

**Key Points:**
- No locks!
- All atomic loads
- Lifetime tied to guard

## The Put Operation (Writing)

### Fast Path: CAS for Empty Bins

```rust
if bin.is_null() {
    let node = Owned::new(BinEntry::Node(Node {
        hash,
        key,
        value: Atomic::new(value),
        next: Atomic::null(),
        lock: Mutex::new(()),
    }));
    
    match table.cas_bin(bin_i, bin, node, &guard) {
        Ok(_) => {
            self.add_count(1, 0);
            return;
        }
        Err(changed) => {
            // Retry
        }
    }
}
```

### Slow Path: Lock and Modify

```rust
BinEntry::Node(head) => {
    let _guard = head.lock.lock();
    
    // Verify head still valid
    let current = table.bin(bin_i, &guard);
    if current.as_raw() != bin.as_raw() {
        continue;
    }
    
    // Walk the linked list
    // Update or append as needed
}
```

---

# 6. Locking Strategy

## Per-Bin Locking

```
Map:
┌─────┬─────┬─────┬─────┐
│Bin 0│Bin 1│Bin 2│Bin 3│
└──┬──┴──┬──┴──┬──┴──┬──┘
   │     │     │     │
   ▼     ▼     ▼     ▼
 Lock  Lock  Lock  Lock
```

**Benefits:**
- Threads on different bins don't block each other
- Scales with number of bins
- Contention only when hashing collides

## Lock Verification After Acquisition

```rust
// 1. Load bin head
let bin = table.bin(bin_i, guard);

// 2. Lock the head
let _guard = head.lock.lock();

// 3. RE-CHECK still the head
let current = table.bin(bin_i, guard);
if current.as_raw() != bin.as_raw() {
    continue;  // Changed! Retry
}

// 4. Now safe to modify
```

---

# 7. The Resizing Process

## When to Resize

```rust
// size_ctl when positive = resize threshold
capacity = 16 bins
size_ctl = 12 (0.75 * 16)

count = 11 → don't resize
count = 12 → trigger resize!
```

## Initializing the New Table

```rust
let new_table = Owned::new(Table {
    bins: vec![Atomic::null(); n << 1].into_boxed_slice(),
});

self.next_table.swap(new_table, Ordering::SeqCst, guard);
self.transfer_index.store(n, Ordering::SeqCst);
```

## Claiming Bins to Transfer

```
transfer_index = 16
stride = 4

Thread A claims: [12, 16)  → transfer_index = 12
Thread B claims: [8, 12)   → transfer_index = 8
Thread C claims: [4, 8)    → transfer_index = 4
```

## The Run Optimization

When resizing from 8 → 16 bins:
- If `hash & 8 == 0` → stays at index i
- If `hash & 8 == 1` → moves to index i+8

**Optimization:** Consecutive nodes going to same destination can be moved as a unit!

---

# 8. Implementation Details

## Hashing Strategy

```rust
fn hash<K: Hash>(&self, key: &K) -> u64 {
    let mut hasher = self.build_hasher.build_hasher();
    key.hash(&mut hasher);
    hasher.finish()
}
```

## Bin Selection Algorithm

```rust
fn bin(&self, hash: u64) -> usize {
    let n = self.bins.len();  // Power of 2
    let mask = (n - 1) as u64;
    (hash & mask) as usize
}
```

**Why it works:**
```
n = 8 = 0b1000
mask = 7 = 0b0111

hash = 0b10110101
mask = 0b00000111
     & ───────────
       0b00000101 = 5
```

## Safety Arguments for Unsafe Code

Every `unsafe` block needs justification:

```rust
// SAFETY: We loaded this under current guard,
// guard is still alive, therefore pointer is valid
let node = unsafe { shared.deref() };
```

---

# 9. Java vs Rust Differences

## Explicit Memory Ordering

### Java (Implicit)
```java
volatile int x;  // Automatically acquire/release
x = 5;           // Release
int y = x;       // Acquire
```

### Rust (Explicit)
```rust
x.store(5, Ordering::Release);
let y = x.load(Ordering::Acquire);
```

## Explicit Lifetime Tracking

```rust
pub fn get<'g>(&self, key: &K, guard: &'g Guard) 
    -> Option<Shared<'g, V>>
//                   ^^
//          Lifetime tied to guard
```

## Type System Enforces Thread Safety

```rust
// HashMap: not thread-safe
impl<K, V> !Sync for HashMap<K, V> {}

// FlurryHashMap: thread-safe
unsafe impl<K, V, S> Sync for FlurryHashMap<K, V, S>
where
    K: Send + Sync,
    V: Send + Sync,
{}
```

## No Garbage Collector

**Java:** 1 line
```java
old = swap(new);  // GC handles cleanup
```

**Rust:** Complex
```rust
let old = swap(new);
// TODO: Queue for deferred destruction
// Track epochs, ensure no readers, free later
```

---

# 10. Current Status & Future Work

## What's Implemented ✓

- [x] Basic structure
- [x] Get (reading)
- [x] Put (writing)
- [x] Per-bin locking
- [x] Resize detection
- [x] Basic bin transfer
- [x] Run optimization

## Not Yet Implemented ❌

**Critical:**
- [ ] Garbage collection (currently leaking!)
- [ ] Moved entry forwarding
- [ ] Help transfer method

**Optimizations:**
- [ ] Tree bins (for many collisions)
- [ ] Counter cells (reduce contention)
- [ ] Better memory ordering

## Required Trait Bounds

```rust
K: Clone + Hash + Eq + Send + Sync
V: Send + Sync
S: BuildHasher + Send + Sync
```

---

# 11. Testing Strategy

## Basic Operations

```rust
#[test]
fn test_insert_and_get() {
    let map = FlurryHashMap::new();
    let guard = crossbeam_epoch::pin();
    
    map.insert("key", "value");
    assert_eq!(map.get(&"key", &guard).unwrap(), "value");
}
```

## Concurrent Operations

```rust
#[test]
fn test_concurrent_inserts() {
    let map = Arc::new(FlurryHashMap::new());
    
    let handles: Vec<_> = (0..10).map(|i| {
        let map = Arc::clone(&map);
        thread::spawn(move || {
            for j in 0..100 {
                map.insert(format!("key-{}-{}", i, j), j);
            }
        })
    }).collect();
    
    for h in handles {
        h.join().unwrap();
    }
    
    // Verify all inserted
}
```

---

# 12. Key Takeaways

## Core Insights

### 1. Lock-Free ≠ Simple
Lock-free data structures are **harder** than locked ones:
- Must reason about all thread interleavings
- Memory ordering becomes critical
- Debugging is much harder

### 2. Garbage Collection Matters
GC isn't "just" convenience - it solves hard problems automatically.

### 3. Type System as Documentation
```rust
pub fn get<'g>(..., guard: &'g Guard) -> Option<Shared<'g, V>>
```
This signature encodes invariants that Java keeps in docs.

### 4. Unsafe Requires Proof
Every `unsafe` block needs a safety argument proving correctness.

### 5. Performance is Subtle
Must benchmark - theory ≠ practice.

## Design Patterns

### 1. Per-Resource Locking
Lock per bin, not per map → better scalability

### 2. Fast Path / Slow Path
```rust
if bin.is_null() {
    // FAST: CAS
} else {
    // SLOW: Lock
}
```

### 3. Cooperative Algorithms
Multiple threads help with expensive operations (resize).

### 4. Epoch-Based Reclamation
Track thread activity to know when safe to free memory.

---

# 13. Appendices

## Glossary

**Acquire:** Memory ordering - load synchronizes with prior Release

**CAS (Compare-And-Swap):** Atomic: if value is X, set to Y

**Epoch:** Time period tracked for memory reclamation

**Guard:** Token proving thread is active in an epoch

**Happens-Before:** Ordering ensuring visibility between threads

**Node:** Entry in bin's linked list with key-value

**Release:** Memory ordering - store synchronizes with later Acquire

**Run:** Consecutive nodes going to same destination during resize

**SeqCst:** Strongest memory ordering, total order

**Shared:** Borrowed pointer valid during guard's lifetime

## Key Structures

```rust
pub struct FlurryHashMap<K, V, S> {
    table: Atomic<Table<K, V>>,
    count: AtomicUsize,
    size_ctl: AtomicIsize,
    transfer_index: AtomicIsize,
    next_table: Atomic<Table<K, V>>,
    build_hasher: S,
}

struct Table<K, V> {
    bins: Box<[Atomic<BinEntry<K, V>>]>,
}

enum BinEntry<K, V> {
    Node(Node<K, V>),
    Moved { next_table: *const Table<K, V> },
}

struct Node<K, V> {
    hash: u64,
    key: K,
    value: Atomic<V>,
    next: Atomic<BinEntry<K, V>>,
    lock: Mutex<()>,
}
```

## Further Reading

### Rust Concurrency
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [Crossbeam Documentation](https://docs.rs/crossbeam/)

### Concurrent Data Structures
- Java ConcurrentHashMap source code
- Art of Multiprocessor Programming

### Jon Gjengset's Resources
- [YouTube Channel](https://www.youtube.com/c/JonGjengset)
- [Rust for Rustaceans](https://nostarch.com/rust-rustaceans)

---

## About This Document

**Source:** Live coding stream by Jon Gjengset  
**Stream Duration:** ~5.5 hours  
**Topic:** Porting Java's ConcurrentHashMap to Rust  
**Project Name:** Flurry  
**Status:** Work in progress  

**Note:** This is an educational resource created from the stream transcript.

---

*End of Document*