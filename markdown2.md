# Concurrent HashMap in Rust - Part 2: Complete Technical Walkthrough

**Stream:** Yet Another Rust Live Stream - Concurrent HashMap Part 2  
**Duration:** ~5 hours 20 minutes  
**Instructor:** Jon Gjengset  
**Topic:** Porting Java's ConcurrentHashMap to Rust  
**Complexity Level:** Advanced  

---

## Table of Contents

1. [Introduction and Session Overview](#1-introduction-and-session-overview)
2. [Architecture Recap: How the Concurrent HashMap Works](#2-architecture-recap-how-the-concurrent-hashmap-works)
3. [Completing the Help Transfer Function](#3-completing-the-help-transfer-function)
4. [The Garbage Collection Problem](#4-the-garbage-collection-problem)
5. [Epoch-Based Memory Reclamation](#5-epoch-based-memory-reclamation)
6. [Implementing Deferred Destruction](#6-implementing-deferred-destruction)
7. [Implementing Drop for Table and HashMap](#7-implementing-drop-for-table-and-hashmap)
8. [Type System Challenges: Merging Node and BinEntry](#8-type-system-challenges-merging-node-and-binentry)
9. [Safety Arguments for Unsafe Code](#9-safety-arguments-for-unsafe-code)
10. [Compilation Journey and Bug Fixes](#10-compilation-journey-and-bug-fixes)
11. [Key Takeaways and Lessons Learned](#11-key-takeaways-and-lessons-learned)
12. [Next Steps](#12-next-steps)

---

## 1. Introduction and Session Overview

### Context

This is Part 2 of porting Java's `ConcurrentHashMap` to Rust. The first stream (6 hours) covered:
- Basic HashMap structure
- Read operations (`get`)
- Write operations (`put`)
- Started table resizing logic

### Goals for This Session

1. **Complete resize logic** - Finish the `help_transfer` function that enables cooperative resizing
2. **Implement garbage collection** - Without Java's GC, manually manage memory reclamation
3. **Add all safety proofs** - Document why every `unsafe` block is actually safe
4. **Achieve compilation** - Fix all type errors and borrow checker issues

### The Fundamental Challenge

**Java approach:**
```java
// Just overwrite - GC handles cleanup
oldValue = newValue; // Old value will be collected when unreachable
```

**Rust challenge:**
```rust
// Must manually determine when safe to free
let old = atomic.swap(new);
// When can we drop `old`? Other threads might still be reading it!
```

In concurrent code, we can't immediately drop old values because other threads might still have pointers to them. This session solves this problem.

---

## 2. Architecture Recap: How the Concurrent HashMap Works

### Visual Overview

```
HashMap Table (array of atomic pointers)
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │
└──│──┴─────┴──│──┴─────┴─────┴─────┘
   │           │
   ▼           ▼
 Node        Node → Node → Node
 (k1,v1)     (k2,v2)  (k3,v3)  (k4,v4)
```

### Key Design Principles

#### 1. Lock-Free Empty Bins
```rust
// If bin is empty, use atomic compare-and-swap
if bin.is_null() {
    bin.compare_and_swap(null, new_node, ordering);
    // No lock needed!
}
```

#### 2. Per-Bin Locking
```rust
// If bin not empty, lock the head node
let head = bin.load();
let _guard = head.lock.lock();
// Now safe to modify the linked list
```

#### 3. Forwarding Nodes During Resize

When resizing, old bins get replaced with special "moved" nodes:

```
Old Table (size 8)          New Table (size 16)
┌─────┬─────┬─────┐        ┌─────┬─────┬─────┬─────┐
│MOVED│MOVED│ ... │        │Node │Node │ ... │ ... │
└──│──┴──│──┴─────┘        └─────┴─────┴─────┴─────┘
   │     │                      ▲
   └─────┴──────────────────────┘
   (both point to new table)
```

Readers encountering a MOVED node follow it to the new table. This allows concurrent reads during resize!

#### 4. Cooperative Resizing

Instead of one thread doing all the work:
```rust
// Thread doing insert finds resize in progress
if bin.is_moved() {
    help_transfer(old_table, new_table);
    // Help move bins, then retry insert
}
```

Multiple threads help transfer bins, speeding up the resize.

### The Hash Distribution Trick

When doubling table size from `n` to `2n`:

```
Hash % 8:  Items hash to bucket 7
           ↓
Hash % 16: Items split between bucket 7 and bucket 15

Example:
7 % 8 = 7,   7 % 16 = 7   (stays in bucket 7)
15 % 8 = 7,  15 % 16 = 15 (moves to bucket 15)
23 % 8 = 7,  23 % 16 = 7  (stays in bucket 7)
```

The high bit determines if an item stays or moves. This enables efficient "run" detection - contiguous sequences that move together.

---

## 3. Completing the Help Transfer Function

### The Purpose

When a thread encounters a MOVED node during `put()`, it should help with the ongoing resize:

```rust
BinEntry::Moved(next_table) => {
    // Don't just wait - help!
    let new_table = self.help_transfer(table, next_table, guard);
    // Then retry insertion in new table
}
```

### Implementation

```rust
fn help_transfer<'g>(
    &self,
    table: Shared<'g, Table<K, V>>,
    next_table: Shared<'g, Table<K, V>>,
    guard: &'g Guard,
) -> Shared<'g, Table<K, V>> {
    // Safety checks
    if table.is_null() || next_table.is_null() {
        return table;
    }
    
    let rs = resize_stamp(table.bins.len());
    
    // Only help if this is the current resize
    while next_table == self.next_table.load(guard, SeqCst) 
        && table == self.table.load(guard, SeqCst) {
        
        let sc = self.size_ctl.load(guard, SeqCst);
        
        // Check if we should stop helping
        if sc >= 0 || sc == rs + MAX_RESIZERS || 
           sc == rs + 1 || transfer_index <= 0 {
            break;
        }
        
        // Try to register as a helper
        if self.size_ctl.compare_and_swap(sc, sc + 1, SeqCst, guard) == sc {
            // We're registered! Do the transfer
            self.transfer(table, next_table, guard);
            break;
        }
    }
    
    next_table
}
```

### Why Multiple Table Pointers?

The code tracks both:
- **`self.next_table`**: The global target table
- **`moved.next_table`**: Where this specific bin moved to

This handles cascading resizes:

```
Thread A reads old_table → moved → middle_table
Thread B triggers another resize: middle_table → newest_table

Thread A's moved node still points to middle_table
But self.next_table now points to newest_table
```

---

## 4. The Garbage Collection Problem

### The Core Issue

Consider this sequence:

```rust
// Thread A
let value = node.value.load(guard);  // Gets pointer to "old_value"
// ... context switch ...

// Thread B  
node.value.swap(new_value);          // Replaces with "new_value"
drop(old_value);                     // Frees memory

// Thread A resumes
println!("{}", value);               // Use-after-free! 💥
```

Thread A has a pointer to freed memory. In a garbage-collected language, this is prevented automatically. In Rust, we must ensure this never happens.

### Why This Is Hard

We need to guarantee:
1. **No premature freeing**: Don't free while someone might use it
2. **No memory leaks**: Eventually free everything
3. **Lock-free performance**: Don't use heavy synchronization

The solution: **Epoch-based memory reclamation**

---

## 5. Epoch-Based Memory Reclamation

### The Core Idea

Think of time divided into epochs:

```
Epoch 1      Epoch 2      Epoch 3
  │            │            │
  ├─────────┤  ├─────────┤  ├────────→
  
Thread A: ──────●══════●─────────────
Thread B: ────────────●═══════●──────
Thread C: ───────────────────●═══●───

● = pin epoch    ═ = holding pin
```

**Rules:**
1. Before accessing shared data, pin the current epoch
2. Mark garbage for collection in current epoch
3. Only free epoch N's garbage after all threads unpinned epoch N

### How Crossbeam Implements This

```rust
use crossbeam_epoch::{self as epoch, Guard};

// Pin the epoch
let guard = epoch::pin();

// Load data - safe while guard held
let node = bin.load(guard);

// Use the data
let value = unsafe { node.deref() }.value;

// Mark old data as garbage
unsafe {
    guard.defer_destroy(old_node);
}

// Drop guard - allows epoch to advance
drop(guard);
```

### The Safety Guarantee

```rust
// Thread A pins epoch 1
let guard = epoch::pin();  // Epoch 1 now can't advance
let ptr = shared.load(guard);

// Thread B marks something as garbage
guard.defer_destroy(old_value);  // Marked for epoch 1

// old_value won't be freed until:
// 1. All threads unpin epoch 1
// 2. Epoch advances to 2
// 3. Maybe even later (implementation detail)

// Thread A's pointer remains valid as long as guard is held
let data = unsafe { ptr.deref() };  // Safe!
```

### Why This Works

The key insight:

> If you read a pointer while epoch N is pinned, and that pointer gets marked as garbage, it will be marked for epoch N. It won't be freed until epoch N+1, which can't happen while you're still pinned to epoch N.

---

## 6. Implementing Deferred Destruction

### Where Garbage Is Generated

The code identifies three main sources of garbage:

#### 1. Value Replacement

```rust
// In put() when replacing an existing value
let old_val = node.value.swap(new_value, SeqCst, guard);

// Safety: No thread executing after this can get a reference to old_val
// because we just swapped it out. Threads that got it before are
// protected by their guard.
unsafe {
    guard.defer_destroy(old_val);
}
```

#### 2. Table Replacement After Resize

```rust
// In transfer() when resize completes
let old_table = self.table.swap(next_table, SeqCst, guard);

// Safety: Only accessible via self.table, which we just swapped.
// Forwarding nodes in even older tables point here, but to reach
// those, you must go through self.table first.
unsafe {
    guard.defer_destroy(old_table);
}
```

#### 3. Node Replacement During Transfer

```rust
// When moving bins during resize
while !p.is_null() {
    let node = unsafe { p.deref() };
    let next = node.next.load(guard, SeqCst);
    
    // Create new node in new table
    let new_node = /* ... */;
    
    // Old node is now garbage
    unsafe {
        guard.defer_destroy(p);
    }
    
    p = next;
}
```

### The Safety Argument Template

For every `defer_destroy`, we must prove:

**Property 1: No future access**
> No thread executing after this line can get a reference to the garbage

**Property 2: Old references protected**
> Threads that already have references must be in current or earlier epochs

**Property 3: Epoch protection**
> Garbage won't be freed until next epoch, after those threads unpin

Example safety comment:
```rust
unsafe {
    guard.defer_destroy(now_garbage);
}
// SAFETY: 
// 1. now_garbage is no longer accessible (we swapped it for moved node)
// 2. Any thread with existing reference got it while an epoch was pinned
// 3. That epoch ≤ current epoch (since we're pinning current)
// 4. Garbage won't be freed until next epoch
// 5. Therefore, all existing references remain valid until their guards drop
```

---

## 7. Implementing Drop for Table and HashMap

### The Table Drop

Tables should only be dropped when:
1. Resize completed and it's the old table
2. The entire HashMap is being dropped

For case 1, all bins should already be MOVED nodes. For case 2, we'll handle it in the HashMap's Drop.

```rust
impl<K, V> Drop for Table<K, V> {
    fn drop(&mut self) {
        // In debug mode, verify all bins are null or moved
        #[cfg(debug_assertions)]
        {
            let guard = unsafe { epoch::unprotected() };
            for bin_idx in 0..self.bins.len() {
                let bin = self.bins[bin_idx].load(guard, SeqCst);
                assert!(bin.is_null() || matches!(
                    unsafe { bin.deref() },
                    BinEntry::Moved(_)
                ));
            }
        }
        
        // Bins should have been moved or freed by whoever is dropping the table
        // If dropped due to resize: all bins are MOVED
        // If dropped due to map drop: map's Drop impl cleaned up
    }
}
```

### The HashMap Drop

This is more complex - we need to walk all bins and clean up:

```rust
impl<K, V> Drop for HashMap<K, V> {
    fn drop(&mut self) {
        // We have &mut self, so no other threads can access
        let guard = unsafe { epoch::unprotected() };
        
        // Verify no resize in progress
        assert!(self.next_table.load(guard, SeqCst).is_null());
        
        let table = self.table.load(guard, SeqCst);
        if table.is_null() {
            return;
        }
        
        let table = unsafe { table.into_owned() };
        
        // Walk each bin
        for bin_idx in 0..table.bins.len() {
            let bin = table.bins[bin_idx].swap(Shared::null(), SeqCst, guard);
            
            if bin.is_null() {
                continue;
            }
            
            match unsafe { bin.into_owned() }.into_box() {
                BinEntry::Moved(moved) => {
                    // Just drop the moved node
                    drop(moved);
                }
                BinEntry::Node(mut head) => {
                    // Walk the linked list
                    loop {
                        // Drop the value
                        let value = head.value.swap(Shared::null(), SeqCst, guard);
                        unsafe {
                            guard.defer_destroy(value);
                        }
                        
                        // Move to next node
                        let next = head.next.swap(Shared::null(), SeqCst, guard);
                        if next.is_null() {
                            break;
                        }
                        
                        // Drop current, move to next
                        drop(head);
                        head = unsafe { next.into_owned() }.into_box();
                    }
                }
            }
        }
    }
}
```

### Why `epoch::unprotected()`?

```rust
let guard = unsafe { epoch::unprotected() };
```

This creates a "dummy" guard that provides no epoch protection. It's safe here because:

1. We have `&mut self` - exclusive access
2. No other threads can access the map
3. We just want the guard for the API, not protection
4. We're dropping everything immediately, not deferring

---

## 8. Type System Challenges: Merging Node and BinEntry

### The Problem

Initially, the code had:

```rust
struct Node<K, V> {
    hash: u64,
    key: K,
    value: Atomic<V>,
    next: Atomic<Node<K, V>>,  // Points to Node
    lock: Mutex<()>,
}

enum BinEntry<K, V> {
    Node(Node<K, V>),
    Moved(Shared<Table<K, V>>),
}

// Bins store:
bins: Vec<Atomic<BinEntry<K, V>>>
```

This seems fine, but there's a critical issue during resize:

### The Issue: Type Mismatch

```rust
// During transfer, we find a "run" - consecutive nodes going to same bin
let run: Shared<Node<K, V>> = /* ... */;

// We want to make this the head of the new bin
new_table.bins[i].store(run, SeqCst);  // ❌ Type error!
// Expected: Atomic<BinEntry<K, V>>
// Found:    Shared<Node<K, V>>
```

The head needs to be `BinEntry`, but the run is `Node`. They're different types!

### Why This Is Hard

Three possible solutions, all with tradeoffs:

#### Option 1: Keep Current Types (Red Solution)
```rust
// Every next pointer could be Node OR Moved
struct Node {
    next: Atomic<BinEntry>,  // ❌ Must match on every access
}

// Problem: We know next is always Node, not Moved!
match node.next {
    BinEntry::Node(n) => { /* use n */ },
    BinEntry::Moved(_) => unreachable!(),  // Runtime check
}
```

**Downside:** Lost type safety, runtime unreachables everywhere

#### Option 2: Double Indirection
```rust
enum BinEntry {
    Node(Atomic<Node>),  // Extra heap allocation
    Moved(Shared<Table>),
}

struct Node {
    next: Atomic<Node>,
}
```

**Downside:** Every access requires two pointer dereferences, cache-unfriendly

#### Option 3: Merge Types (Chosen)
```rust
enum BinEntry {
    Node {
        hash: u64,
        key: K,
        value: Atomic<V>,
        next: Atomic<BinEntry>,  // Can point to Node or Moved
        lock: Mutex<()>,
    },
    Moved(Shared<Table>),
}
```

**Downside:** Must pattern match to access node fields, slightly larger enum

### Implementation Choice

The stream chooses **Option 3** for several reasons:

1. **No extra allocation overhead** - single heap allocation per node
2. **No double indirection** - one pointer deref to access node
3. **Type system prevents some errors** - can't accidentally use Moved as Node
4. **Runtime checks only where needed** - matching is required anyway

The tradeoff is more verbose code:

```rust
// Before
let value = node.value;

// After  
let value = match bin {
    BinEntry::Node { value, .. } => value,
    BinEntry::Moved(_) => unreachable!(),
};
```

But helper methods can reduce this:

```rust
impl BinEntry {
    fn as_node(&self) -> Option<&NodeFields> {
        match self {
            BinEntry::Node { hash, key, value, next, lock } => {
                Some(&NodeFields { hash, key, value, next, lock })
            }
            BinEntry::Moved(_) => None,
        }
    }
}

// Usage
let node = bin.as_node().unwrap();
let value = node.value;
```

---

## 9. Safety Arguments for Unsafe Code

### The Shared Dereference Problem

Crossbeam's `Shared<T>` is a thin wrapper around a raw pointer:

```rust
pub struct Shared<'g, T> {
    ptr: *const T,
    _marker: PhantomData<&'g T>,
}

impl<T> Shared<'_, T> {
    pub unsafe fn deref(&self) -> &T {
        &*self.ptr  // Raw pointer deref - unsafe!
    }
}
```

**Why unsafe?** The pointer might:
1. Point to freed memory
2. Point to uninitialized memory
3. Be null
4. Race with concurrent writes

Every dereference needs a safety proof.

### The Safety Template

For every `unsafe { shared.deref() }`:

```rust
let node = unsafe { bin.deref() };
// SAFETY:
// 1. bin is non-null (checked above)
// 2. bin was loaded while guard was held
// 3. bin won't be freed until next epoch
// 4. we still hold the guard, so next epoch can't arrive
// 5. therefore, bin remains valid
```

### Common Safety Patterns

#### Pattern 1: Loaded While Guarded

```rust
let guard = epoch::pin();
let ptr = atomic.load(guard, SeqCst);

// Safe because:
// - Loaded in current epoch
// - Guard prevents epoch advancement
// - defer_destroy won't free until next epoch
let data = unsafe { ptr.deref() };
```

#### Pattern 2: Swapped After Guard

```rust
let guard = epoch::pin();
let old = atomic.swap(new, SeqCst, guard);

// Safe to destroy because:
// - No future thread can get `old` (we swapped it out)
// - Threads that got it before are in ≤ current epoch
// - Won't free until next epoch
unsafe { guard.defer_destroy(old); }
```

#### Pattern 3: Owned by Guard

```rust
let guard = epoch::pin();
let table = self.table.load(guard, SeqCst);
let bin = table.bins[i].load(guard, SeqCst);

// Safe because:
// - table loaded while guard held -> valid
// - bins owned by table -> also valid
// - both protected by same guard
let data = unsafe { bin.deref() };
```

#### Pattern 4: Exclusive Ownership

```rust
impl Drop for HashMap {
    fn drop(&mut self) {
        let guard = unsafe { epoch::unprotected() };
        // Safe because:
        // - We have &mut self (exclusive access)
        // - No other threads exist
        // - unprotected() just provides API compatibility
    }
}
```

### The Tricky Case: Moved Nodes

The most complex safety argument:

```rust
// In put(), we might follow a moved node
let table = self.table.load(guard, SeqCst);
match bin {
    BinEntry::Moved(next_table) => {
        // next_table is a raw pointer - safe to deref?
        let next = unsafe { next_table.deref() };
    }
}
```

**The argument:**

```
1. We loaded `table` while `guard` was held
2. We got `bin` from `table.bins[i]`, also while guarded
3. `bin` is a Moved node containing `next_table` pointer
4. For `next_table` to be freed:
   a. The resize must complete
   b. `next_table` must be swapped out of `self.table`
   c. An epoch must pass
5. But we're in the current epoch (holding guard)
6. So `next_table` is still valid
```

The stream leaves this marked as `FIXME` to revisit the exact wording.

### Safety Documentation Strategy

The code uses this format consistently:

```rust
// SAFETY:
// [Property being guaranteed]
// [Why that property holds]
// [How it relates to epoch protection]
unsafe {
    // unsafe operation
}
```

This makes auditing easier and explains the reasoning to future readers.

---

## 10. Compilation Journey and Bug Fixes

### Phase 1: Basic Type Errors (Errors ~50)

Initial compilation reveals typical porting errors:

```rust
// Error: no method `finish` on `DefaultHasher`
hasher.finish()  // ✓ Correct

// Error: wrong number of type arguments
table.bins.len()  // Was: table.len()

// Error: mismatched types i64 vs usize
resize_stamp(n)  // Added: as isize

// Error: no variant `Release` 
Ordering::Release  // Was: Ordering::release (typo)
```

### Phase 2: Guard Parameters (Errors ~30)

Many methods need guards for epoch protection:

```rust
// Before
fn transfer(&self, table: Shared<Table>) { }

// After  
fn transfer<'g>(&self, table: Shared<'g, Table>, guard: &'g Guard) { }
```

The `'g` lifetime ties the `Shared` to the guard's lifetime.

### Phase 3: Atomic Operations (Errors ~20)

Crossbeam's atomics have specific signatures:

```rust
// load(guard, ordering)
let val = atomic.load(guard, SeqCst);

// compare_and_swap(current, new, ordering, guard)
atomic.compare_and_swap(old, new, SeqCst, guard);

// swap(new, ordering, guard)  
atomic.swap(new, SeqCst, guard);
```

Many errors from wrong parameter order.

### Phase 4: Deref Everywhere (Errors ~15)

After merging Node into BinEntry:

```rust
// Before (when Node was separate)
let value = node.value;

// After (BinEntry enum)
let value = match bin {
    BinEntry::Node { value, .. } => value,
    BinEntry::Moved(_) => unreachable!(),
};

// Or with helper
let value = bin.as_node().unwrap().value;
```

Added `as_node()` helper to reduce boilerplate.

### Phase 5: Ownership Issues (Errors ~8)

```rust
// Error: can't move out of borrowed content
for bin in &table.bins {
    drop(bin);  // ❌ bin is &Atomic<BinEntry>
}

// Fix: use into_owned or swap
let bin = table.bins[i].swap(Shared::null(), SeqCst, guard);
let owned = unsafe { bin.into_owned() };
drop(owned);  // ✓
```

### Phase 6: Clone Requirement

```rust
// Error: K is not cloneable
let new_key = key;  // Trying to use key in new node

// During transfer, we clone old nodes
// Old node still exists (may be read by other threads)
// New node needs its own copy of key

// Solution: add Clone bound
impl<K: Clone, V> HashMap<K, V> { }
```

This is unfortunate but necessary - we can't move the key because the old node might still be accessed.

### Phase 7: Borrow Checker Battles (Final errors)

```rust
// Error: cannot borrow as mutable
let key = &node.key;
// ... later ...
let new_node = Node { key, .. };  // Borrow still active

// Fix: limit borrow scope
let hash = {
    let node = unsafe { bin.deref() };
    node.hash
};
// Borrow ends here
let new_node = Node { hash, .. };  // ✓
```

### Final Compilation

After 5+ hours:

```rust
$ cargo check
   Compiling flurry v0.1.0
    Finished in 42.3s
```

**Success!** The code compiles with:
- 0 errors
- A few warnings (unused variables, etc.)
- ~20 FIXME comments for next session

---

## 11. Key Takeaways and Lessons Learned

### 1. Unsafe Rust Requires Rigorous Reasoning

**Key Insight:** Unsafe code isn't "do whatever you want" - it's "you must prove the invariants the compiler can't check."

Every unsafe block in this code has a comment explaining:
- What invariant is being relied upon
- Why that invariant holds
- How epoch protection ensures safety

This is not optional - it's the difference between correct code and undefined behavior.

### 2. Epoch-Based Reclamation Is Powerful But Requires Discipline

The epoch system provides:
- ✅ Lock-free memory reclamation
- ✅ Better performance than reference counting
- ✅ Simpler than hazard pointers

But requires:
- ⚠️ Careful guard management (don't hold too long)
- ⚠️ Understanding when pins can be dropped
- ⚠️ Proving that guards protect what you think they protect

### 3. Type System Limitations in Concurrent Code

Some invariants can't be expressed in Rust's type system:

```rust
// We know: node.next is always BinEntry::Node, never Moved
// Type system: node.next is BinEntry (could be either)

// Result: runtime checks
match node.next {
    BinEntry::Node(n) => n,
    BinEntry::Moved(_) => unreachable!(),
}
```

The alternative (double indirection) is worse for performance, so we accept runtime checks.

### 4. Porting Concurrent Code Is Not Straightforward

Java code:
```java
volatile Node<K,V> next;
// JMM guarantees visibility
```

Rust equivalent requires:
- Choosing the right atomic type (`Atomic<Shared<T>>`)
- Choosing ordering (SeqCst for now, could relax later)
- Managing epochs for memory safety
- Explicit safety proofs

### 5. Performance Tradeoffs Are Everywhere

Examples from this stream:

**Clone requirement:**
```rust
impl<K: Clone, V> HashMap<K, V>
```
- Pro: Simpler implementation
- Con: Extra allocations during resize
- Alternative: Arc<K> (extra indirection on every access)

**SeqCst ordering:**
```rust
atomic.load(guard, SeqCst)
```
- Pro: Easy to reason about, definitely safe
- Con: Stronger than necessary (could use Acquire/Release)
- Optimization: Future work

**Merged BinEntry type:**
```rust
enum BinEntry { Node { .. }, Moved(..) }
```
- Pro: No double indirection
- Con: Slightly larger size, more pattern matching

### 6. Safety Comments Are Code

The safety comments aren't just documentation - they're essential reasoning:

```rust
unsafe { guard.defer_destroy(old_value); }
// SAFETY: No thread executing after this line can get a reference
// to old_value (we swapped it out). Threads that got it before
// are in ≤ current epoch, and defer_destroy won't free until
// next epoch, so their references remain valid.
```

If you can't write this comment convincingly, the code is probably wrong.

---

## 12. Next Steps

### What Works Now

✅ Code compiles!  
✅ Basic structure complete  
✅ Insert operation implemented  
✅ Get operation implemented  
✅ Resize with cooperative helping  
✅ Garbage collection via epochs  

### What's Missing

❌ **Testing** - No tests have been run yet  
❌ **Remove operation** - Not implemented  
❌ **Iterator** - Complex due to epoch management  
❌ **TreeBin** - Java uses red-black trees for large bins  
❌ **Counter** - Sharded counter for size()  
❌ **One FIXME safety proof** - Moved node deref  

### Plan for Next Stream

1. **Write tests** - Port Java's test suite
2. **Fix bugs** - Definitely has bugs
3. **Implement remove** - Complete the basic API
4. **Performance testing** - Compare to other implementations
5. **Documentation** - Public API docs

### Recommended Learning Path

If you want to understand this code:

1. **Read about epochs** - Crossbeam's epoch documentation
2. **Study the Java code** - Available in JDK source
3. **Watch both streams** - Part 1 for basics, Part 2 for GC
4. **Try the code** - Clone the repo and experiment
5. **Read the safety comments** - Understand each unsafe block

### Resources

- **Crossbeam epoch docs:** https://docs.rs/crossbeam-epoch
- **Java ConcurrentHashMap:** OpenJDK source code
- **Epoch paper:** "Epoch-Based Reclamation" by Fraser
- **Stream repo:** (mentioned in stream, check description)

---

## Appendix: Code Structure Overview

### File Organization

```
src/
├── lib.rs              # Public API
├── map.rs              # HashMap implementation  
├── node.rs             # BinEntry and Node types
└── table.rs            # Table structure
```

### Key Types

```rust
// The main type
pub struct HashMap<K, V, S = RandomState> {
    table: Atomic<Table<K, V>>,
    next_table: Atomic<Table<K, V>>,
    size_ctl: AtomicIsize,
    // ...
}

// Storage
struct Table<K, V> {
    bins: Box<[Atomic<BinEntry<K, V>>]>,
}

// Entry types  
enum BinEntry<K, V> {
    Node {
        hash: u64,
        key: K,
        value: Atomic<V>,
        next: Atomic<BinEntry<K, V>>,
        lock: Mutex<()>,
    },
    Moved(Shared<Table<K, V>>),
}
```

### Key Operations

```rust
// Insert
pub fn insert(&self, key: K, value: V) -> Option<V>

// Lookup
pub fn get(&self, key: &K) -> Option<&V>

// Internal
fn put(&self, key: K, value: V, no_replacement: bool) -> Option<V>
fn transfer(&self, table: Shared<Table>, next_table: Shared<Table>, guard: &Guard)
fn help_transfer(&self, table: Shared<Table>, next_table: Shared<Table>, guard: &Guard)
fn init_table(&self, guard: &Guard) -> Shared<Table>
```

---

## Conclusion

This stream demonstrates that porting concurrent Java code to Rust is challenging but achievable. The main difficulties are:

1. **No garbage collector** - Must manually prove memory safety
2. **Explicit lifetimes** - Guard management is complex
3. **Type system limitations** - Some invariants need runtime checks
4. **Safety proofs** - Every unsafe block needs justification

However, the result is:
- Memory safe without GC overhead
- Lock-free performance
- Explicit concurrency contracts
- Compiler-verified correctness (for safe code)

The next stream will test whether it actually works!

---

**Document End**

Total Stream Time: ~5 hours 20 minutes  
Lines of Code: ~2000+  
Unsafe Blocks: ~50+  
Safety Comments: ~50+  
Compilation Errors Fixed: ~100+