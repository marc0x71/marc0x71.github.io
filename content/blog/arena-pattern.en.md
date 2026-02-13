+++
title = "Arena Pattern in Rust: Managing Cyclic References with Indices"
date = 2026-02-25
description = "There's a rite of passage every Rust programmer must face: writing a doubly linked list. In C++ it's a beginner exercise. In Python it takes three lines of code. In Rust, it's the moment many beginners hit a brick wall: the borrow checker. Suddenly, simple concepts like 'node A points to B and B points to A' become a nightmare of lifetimes, RefCell, and runtime panics. You feel stupid for not being able to write a data structure that a C programmer could write blindfolded in 1985. The good news? You're not the problem — the mental model is. In this article we'll stop fighting the compiler and learn to solve the problem at its root, using the Arena Pattern."

draft=true

[taxonomies]
tags = ["rust", "pattern"]
+++

There's a rite of passage every Rust programmer must face: writing a doubly linked list.

In C++ it's a beginner exercise. In Python it takes three lines of code. In Rust, it's the moment many beginners hit a brick wall: the borrow checker. Suddenly, simple concepts like "node A points to B and B points to A" become a nightmare of lifetimes, `RefCell`, and runtime panics. You feel stupid for not being able to write a data structure that a C programmer could write blindfolded in 1985.

The good news? You're not the problem — the mental model is. In this article we'll stop fighting the compiler and learn to solve the problem at its root, using the **Arena Pattern**.

---

## But let's start from the beginning

In Python, the doubly linked list we're talking about is written like this:

```python
class Node:
    def __init__(self, value, next=None, prev=None):
        self.value = value
        self.next = next
        self.prev = prev
```

No ownership issues, no lifetimes, no compiler blocking you. References are simply pointers, and the garbage collector takes care of everything.

In Rust too, a **singly linked list** is doable without too much trouble. The classic recursive version with `Box` works:

```rust
enum List<T> {
    Nil,
    Cons(T, Box<List<T>>),
}
```

Ownership is clear: each node owns the next one through `Box`. A linear chain, with no ambiguity. The compiler is satisfied and everything runs smoothly.

But what happens when we want to go **backwards too**?

---

## The wall: the doubly linked list

Let's try the most natural approach — each node has a `next` pointer and a `prev` pointer:

```rust
struct Node<T> {
    value: T,
    next: Option<Box<Node<T>>>,
    prev: Option<Box<Node<T>>>,
}
```

The struct compiles, but `Box` implies exclusive ownership, so the resulting structure is a tree, not a graph. A doubly linked list requires shared ownership or non-owning pointers; with `Box` alone it's not possible to correctly represent bidirectional links without violating the ownership rules.

Let's try references then:

```rust
struct Node<'a, T> {
    value: T,
    next: Option<&'a mut Node<'a, T>>,
    prev: Option<&'a mut Node<'a, T>>,
}
```

Here too the definition compiles, but trying to use it is where everything falls apart — and for deeper reasons than it might seem at first glance. The first problem is technical: you can't have two `&mut` to the same node (one from the predecessor, one from the successor), and the lifetimes become a recursive nightmare.

But there's an even more fundamental problem: **where do the nodes live?** A reference `&'a Node` doesn't own anything — it points to something that must already exist somewhere. And that "somewhere" is the real crux of the matter.

If you allocate them on the stack, you have to declare them all before linking them, making it impossible to build the list dynamically:

```rust
// Each node is a local variable... doesn't scale
let mut node_a = Node { value: 1, next: None, prev: None };
let mut node_b = Node { value: 2, next: None, prev: None };
// How do I link node_b.prev to node_a if I've already taken &mut node_a for next?
```

If you allocate them on the heap with `Box`, you fall back into the exclusive ownership problem: the `Box` owns the node, and you can't have a `prev` reference to something owned by another node's `Box` without violating the borrow checker rules.

It's a logical short-circuit: to use references, the nodes must exist somewhere, but every "somewhere" in Rust has ownership rules that conflict with bidirectional references.

The underlying problem is that the doubly linked list has **bidirectional references**: each node is pointed to from two directions. In Rust, where ownership is exclusive and lifetimes are strict, this is a fundamental contradiction.

---

## The classic solution: `Rc<RefCell<T>>`

Rust's traditional answer to shared and mutable references is the `Rc<RefCell<T>>` pair:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node<T> {
    value: T,
    next: Option<Rc<RefCell<Node<T>>>>,
    prev: Option<Weak<RefCell<Node<T>>>>,
}
```

`Rc` (Reference Counted) allows shared ownership: multiple parts of the code can own the same node. `RefCell` moves borrow checking from compile-time to runtime, allowing interior mutability. `Weak` is a weak version of `Rc` that doesn't contribute to the reference count, needed for the `prev` pointer to avoid reference cycles that would cause memory leaks.

It works, but at what cost? Let's look at a simple operation — inserting a new node after an existing one:

```rust
fn insert_after<T>(node: &Rc<RefCell<Node<T>>>, value: T) {
    let new_node = Rc::new(RefCell::new(Node {
        value,
        next: None,
        prev: None,
    }));

    let mut node_borrow = node.borrow_mut();
    let old_next = node_borrow.next.take();

    // Link new_node -> old_next
    if let Some(ref old_next_rc) = old_next {
        old_next_rc.borrow_mut().prev = Some(Rc::downgrade(&new_node));
    }
    new_node.borrow_mut().next = old_next;

    // Link node -> new_node
    new_node.borrow_mut().prev = Some(Rc::downgrade(node));
    node_borrow.next = Some(new_node);
}
```

For an operation that conceptually is "add a node between two nodes and update four pointers," the code is surprisingly dense. But the real problem isn't the verbosity — it's the hidden trade-offs:

- **Runtime overhead**: every `Rc::clone()` and every `drop` increments/decrements an atomic counter. Every `borrow()` and `borrow_mut()` performs a runtime check.
- **Runtime panics**: if you call `borrow_mut()` while a `borrow()` is already active on the same `RefCell`, the program panics. You've moved a compile-time error to a runtime error — a step backward.
- **Silent memory leaks**: if you forget to use `Weak` for `prev` and use `Rc` instead, you create a reference cycle that will never be deallocated. No warning, no error — just growing memory.
- **Fallibility of `Weak`**: `prev.upgrade()` returns `Option<Rc<...>>` because the node might already have been deallocated. You have to handle this case everywhere.

There's something deeply unsatisfying about all of this. One of Rust's fundamental promises is to free the programmer from the _invisible checklist_ that those coming from C/C++ know well: that constant background of defensive questions ("Is this pointer null?", "Did I free this memory?", "Is this reference still valid?") that drains mental energy that should be devoted to the problem's logic. With `Rc<RefCell<T>>` we're reintroducing an invisible checklist inside Rust: remember to use `Weak` for backward references, don't call `borrow_mut()` while a `borrow()` is active, handle `upgrade()` which can fail. Our energy is back to being spent on defense, not construction. (For a deeper look at this topic, I recommend [Switching from C/C++ to Rust: the invisible checklist](https://marc0x71.github.io/blog/checklist-invisibile/) which explores the difference between the _avoidant muscle memory_ of C++ and the _constructive_ one of Rust.)

Is there a better way?

---

## Changing perspective: the Arena Pattern

Let's take a step back and reflect. The fundamental problem with the doubly linked list isn't the data structure itself — it's that we're asking **each node to manage ownership of its neighbors**. What if we flipped the model? What if **no node owned any other node**?

This is the intuition behind the Arena Pattern: a central structure, the _arena_, owns all the nodes. Nodes don't own each other; they **refer** to one another through simple numeric indices into the arena.

```rust
type NodeId = usize;

struct Node<T> {
    value: T,
    next: Option<NodeId>,
    prev: Option<NodeId>,
}

struct LinkedList<T> {
    arena: Vec<Node<T>>,
    head: Option<NodeId>,
    tail: Option<NodeId>,
}
```

No lifetimes, no `Rc`, no `RefCell`. A `NodeId` is simply a `usize` — an index into the `Vec` serving as the arena. And `usize` is `Copy`, has no lifetime, creates no borrows. It can be passed, copied, and stored anywhere without the borrow checker having anything to say about it.

Let's revisit the `insert_after` operation with this approach:

```rust
impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList {
            arena: Vec::new(),
            head: None,
            tail: None,
        }
    }

    fn push_back(&mut self, value: T) -> NodeId {
        let new_id = self.arena.len();
        self.arena.push(Node {
            value,
            prev: self.tail,
            next: None,
        });

        if let Some(tail_id) = self.tail {
            self.arena[tail_id].next = Some(new_id);
        } else {
            self.head = Some(new_id);
        }

        self.tail = Some(new_id);
        new_id
    }

    fn insert_after(&mut self, node_id: NodeId, value: T) -> NodeId {
        let new_id = self.arena.len();
        let old_next = self.arena[node_id].next;

        self.arena.push(Node {
            value,
            prev: Some(node_id),
            next: old_next,
        });

        self.arena[node_id].next = Some(new_id);

        if let Some(next_id) = old_next {
            self.arena[next_id].prev = Some(new_id);
        } else {
            self.tail = Some(new_id);
        }

        new_id
    }
}
```

The code is linear, readable, and does exactly what it says. No ceremony, no runtime overhead, no risk of panics from `borrow_mut()`. The compiler is happy, and so are we.

---

## Building a generic Arena

Let's extract the pattern into a reusable structure. In its simplest form, an arena is a `Vec` with an allocation method:

```rust
struct Arena<T> {
    items: Vec<T>,
}

impl<T> Arena<T> {
    fn new() -> Self {
        Arena { items: Vec::new() }
    }

    fn alloc(&mut self, value: T) -> usize {
        let id = self.items.len();
        self.items.push(value);
        id
    }

    fn get(&self, id: usize) -> &T {
        &self.items[id]
    }

    fn get_mut(&mut self, id: usize) -> &mut T {
        &mut self.items[id]
    }
}
```

Allocation is amortized O(1) (like a `Vec::push`). Access is O(1) (a direct array lookup). Simple, fast, predictable.

But this arena has an important limitation: it's **append-only**. Once an element is allocated, you can't remove it. For many use cases (a compiler's AST, parser nodes) this is perfectly acceptable: you allocate everything during one phase, use everything, and then discard the entire arena. But for our linked list, where we want to be able to remove nodes, we need something more.

---

## Removal: first approach with a free list

The removal problem is subtler than it seems. The first thing that comes to mind is using `Vec::remove(index)`, but this compacts the vector by shifting all subsequent elements back by one position. The node that was at index 4 is now at 3, the one at 5 is now at 4, and so on — all IDs stored elsewhere become wrong. In a linked list, where each node stores the indices of its neighbors, that would be a disaster.

The alternative is to not actually remove the element, but replace it with a placeholder (`None`), leaving a "hole" in the vector. The indices remain stable, but the holes accumulate and subsequent allocations still go at the end of the `Vec`, without reusing the free space. Over time the arena grows for no reason.

The solution is to maintain a **free list** — an auxiliary vector that keeps track of the indices of free slots:

```rust
struct Arena<T> {
    items: Vec<Option<T>>,
    free_list: Vec<usize>,
}

impl<T> Arena<T> {
    fn new() -> Self {
        Arena {
            items: Vec::new(),
            free_list: Vec::new(),
        }
    }

    fn alloc(&mut self, value: T) -> usize {
        if let Some(index) = self.free_list.pop() {
            self.items[index] = Some(value);
            index
        } else {
            let index = self.items.len();
            self.items.push(Some(value));
            index
        }
    }

    fn remove(&mut self, id: usize) -> Option<T> {
        let value = self.items[id].take();
        if value.is_some() {
            self.free_list.push(id);
        }
        value
    }

    fn get(&self, id: usize) -> Option<&T> {
        self.items.get(id)?.as_ref()
    }
}
```

It works. When you allocate, you first check if there's a free slot in the free list; if so, you reuse it, otherwise you expand the `Vec`. When you remove, you put `None` in the slot and add the index to the free list.

But there's an inelegance: you're maintaining **two separate structures** — the `Vec<Option<T>>` for the data and the `Vec<usize>` for the free slots. The empty slots contain `None`, still taking up space for `Option<T>`, and meanwhile their index is duplicated in the free list. We can do better.

---

## The intrusive free list: a more elegant arena

The key idea: instead of keeping the list of free slots in a separate vector, **encode it directly inside the arena**. Each slot is an enum: it either contains a value (`Occupied`) or is empty and points to the next empty slot (`Vacant`). The empty slots form an intrusive linked list inside the arena itself.

```rust
enum Slot<T> {
    Occupied(T),
    Vacant { next_free: Option<usize> },
}

struct Arena<T> {
    slots: Vec<Slot<T>>,
    head_free: Option<usize>,
    len: usize,
}
```

The space a removed node leaves behind isn't wasted: it contains the pointer to the next free slot. Zero overhead.

A note on style: you might find implementations that use a sentinel value (typically `usize::MAX`) instead of `Option` to signal "no next slot." This is a common approach in C, but it's unidiomatic in Rust. `Option` communicates intent explicitly, is verified by the compiler, and thanks to [niche optimization](https://doc.rust-lang.org/std/option/index.html#representation), an `Option<usize>` doesn't take more memory than a `usize` on many platforms.

### Full implementation

```rust
enum Slot<T> {
    Occupied(T),
    Vacant { next_free: Option<usize> },
}

struct Arena<T> {
    slots: Vec<Slot<T>>,
    head_free: Option<usize>,
    len: usize,
}

impl<T> Arena<T> {
    /// Creates an empty arena
    fn new() -> Self {
        Arena {
            slots: Vec::new(),
            head_free: None,
            len: 0,
        }
    }

    /// Creates an arena with `capacity` pre-allocated slots.
    /// All slots start as Vacant and form a chain.
    fn with_capacity(capacity: usize) -> Self {
        let mut slots = Vec::with_capacity(capacity);
        for i in 0..capacity {
            let next = if i + 1 < capacity { Some(i + 1) } else { None };
            slots.push(Slot::Vacant { next_free: next });
        }
        Arena {
            slots,
            head_free: if capacity > 0 { Some(0) } else { None },
            len: 0,
        }
    }

    /// Allocates a value in the arena, returns its ID.
    fn alloc(&mut self, value: T) -> usize {
        self.len += 1;

        if let Some(index) = self.head_free {
            let next = match self.slots[index] {
                Slot::Vacant { next_free } => next_free,
                Slot::Occupied(_) => unreachable!(),
            };
            self.slots[index] = Slot::Occupied(value);
            self.head_free = next;
            index
        } else {
            let index = self.slots.len();
            self.slots.push(Slot::Occupied(value));
            index
        }
    }

    /// Removes a value from the arena, returning it.
    fn remove(&mut self, id: usize) -> Option<T> {
        let slot = self.slots.get_mut(id)?;
        match std::mem::replace(
            slot,
            Slot::Vacant { next_free: self.head_free },
        ) {
            Slot::Occupied(value) => {
                self.head_free = Some(id);
                self.len -= 1;
                Some(value)
            }
            vacant @ Slot::Vacant { .. } => {
                // It was already empty, restore it
                *slot = vacant;
                None
            }
        }
    }

    /// Immutable access to a value.
    fn get(&self, id: usize) -> Option<&T> {
        match self.slots.get(id)? {
            Slot::Occupied(value) => Some(value),
            Slot::Vacant { .. } => None,
        }
    }

    /// Mutable access to a value.
    fn get_mut(&mut self, id: usize) -> Option<&mut T> {
        match self.slots.get_mut(id)? {
            Slot::Occupied(value) => Some(value),
            Slot::Vacant { .. } => None,
        }
    }

    /// Number of active elements in the arena.
    fn len(&self) -> usize {
        self.len
    }
}
```

### How it works: a visual example

```
Arena::with_capacity(5):

  head_free: Some(0)
  [0: Vacant→1] → [1: Vacant→2] → [2: Vacant→3] → [3: Vacant→4] → [4: Vacant→None]

After alloc("A"), alloc("B"), alloc("C"):

  head_free: Some(3)
  [0: Occ("A")]   [1: Occ("B")]   [2: Occ("C")]   [3: Vacant→4] → [4: Vacant→None]

After remove(1):      slot 1 becomes the new head of the free list

  head_free: Some(1)
  [0: Occ("A")]   [1: Vacant→3] → [3: Vacant→4] → [4: Vacant→None]
                   [2: Occ("C")]

After alloc("D"):     reuses slot 1

  head_free: Some(3)
  [0: Occ("A")]   [1: Occ("D")]   [2: Occ("C")]   [3: Vacant→4] → [4: Vacant→None]
```

Both `alloc` and `remove` are **O(1)**: `alloc` takes from the head of the free list, `remove` inserts at the head. There's never a need to search for a free slot.

---

## The limits: stale IDs

The arena with the intrusive free list is elegant and performant, but it has an important limitation to be aware of. When a slot is removed and then reused, an old index stored somewhere in the code now points to a **completely different object**:

```rust
let mut arena = Arena::new();

let id_carlo = arena.alloc("Carlo");   // id = 0
arena.remove(id_carlo);                // slot 0 → Vacant
let id_diana = arena.alloc("Diana");   // reuses slot 0, id = 0

// ⚠️ id_carlo is still 0, but now points to "Diana"!
println!("{:?}", arena.get(id_carlo)); // Some("Diana"), silent BUG!
```

It's the logical equivalent of a **use-after-free**: the ID is formally valid (the slot is `Occupied`), but it refers to a different object than the original one. No panic, no error — just corrupted data.

For scenarios where removal is frequent and IDs are held for a long time, solutions exist such as the **generational index** (each slot has a "generation" counter that invalidates old IDs) and the **newtype pattern with PhantomData** (to prevent mixing IDs from different arenas at compile-time). But these refinements deserve a dedicated article; for now, it's enough to be aware of the risk and adopt good practices: don't hold onto IDs after a `remove`, or always verify the result of `get`, which returns `Option`.

> **Side note**: the Occupied/Vacant pattern we built isn't limited to traditional in-memory data structures. The same approach applies, for instance, to managing records in shared memory for inter-process communication, where a pre-allocated table with occupied and vacant slots allows O(1) insertions, lookups, and removals in a context where dynamic allocations aren't possible. I used this same pattern in [inmemorytable-rs](https://github.com/marc0x71/inmemorytable-rs), a library I developed for managing typed records in POSIX shared memory.

---

## Putting it all together: the doubly linked list with the Arena

Let's go back to our starting point. With the intrusive free list arena, we can finally write a complete doubly linked list — with insertion and removal — without `Rc`, without `RefCell`, without lifetimes:

```rust
type NodeId = usize;

struct ListNode<T> {
    value: T,
    prev: Option<NodeId>,
    next: Option<NodeId>,
}

struct DoublyLinkedList<T> {
    arena: Arena<ListNode<T>>,
    head: Option<NodeId>,
    tail: Option<NodeId>,
}

impl<T> DoublyLinkedList<T> {
    fn new() -> Self {
        DoublyLinkedList {
            arena: Arena::new(),
            head: None,
            tail: None,
        }
    }

    fn push_back(&mut self, value: T) -> NodeId {
        let node = ListNode {
            value,
            prev: self.tail,
            next: None,
        };
        let id = self.arena.alloc(node);

        if let Some(tail_id) = self.tail {
            self.arena.get_mut(tail_id).unwrap().next = Some(id);
        } else {
            self.head = Some(id);
        }

        self.tail = Some(id);
        id
    }

    fn push_front(&mut self, value: T) -> NodeId {
        let node = ListNode {
            value,
            prev: None,
            next: self.head,
        };
        let id = self.arena.alloc(node);

        if let Some(head_id) = self.head {
            self.arena.get_mut(head_id).unwrap().prev = Some(id);
        } else {
            self.tail = Some(id);
        }

        self.head = Some(id);
        id
    }

    fn remove(&mut self, id: NodeId) -> Option<T> {
        let node = self.arena.get(id)?;
        let prev = node.prev;
        let next = node.next;

        match prev {
            Some(prev_id) => {
                self.arena.get_mut(prev_id).unwrap().next = next;
            }
            None => self.head = next,
        }

        match next {
            Some(next_id) => {
                self.arena.get_mut(next_id).unwrap().prev = prev;
            }
            None => self.tail = prev,
        }

        self.arena.remove(id).map(|node| node.value)
    }

    fn iter(&self) -> impl Iterator<Item = &T> {
        let arena = &self.arena;
        let mut current = self.head;

        std::iter::from_fn(move || {
            let id = current?;
            let node = arena.get(id)?;
            current = node.next;
            Some(&node.value)
        })
    }
}
```

Let's compare it with the `Rc<RefCell<T>>` version:

| Aspect | `Rc<RefCell<T>>` | Arena + indices |
|---|---|---|
| **Link type** | `Option<Rc<RefCell<Node<T>>>>` | `Option<usize>` |
| **Insertion** | `borrow_mut()`, `Rc::new`, `downgrade` | Direct index access |
| **Removal** | `upgrade()`, `Weak` handling | `arena.remove(id)` |
| **Runtime risk** | Panic from double `borrow_mut` | None (returns `Option`) |
| **Overhead** | Reference counting + borrow check | Zero |
| **Cache-friendliness** | Nodes scattered across the heap | Nodes contiguous in the `Vec` |

---

## Beyond the linked list

If the pattern works for a doubly linked list, it works for any structure with complex references. The linked list is the simplest case, but the same approach applies to:

- **Trees with parent pointers**: each node has an `Option<Id>` for its parent and a `Vec<Id>` for its children.
- **Arbitrary graphs**: each node has a `Vec<Id>` for its neighbors, supporting cycles with no issues.
- **ASTs in compilers**: the Rust compiler itself (`rustc`) uses arenas internally for Abstract Syntax Tree nodes.
- **Entity Component Systems**: game engines like Bevy use arenas (via `slotmap` or similar structures) to manage game entities.

The principle is always the same: **the arena owns the data, the indices describe the relationships**.

---

## The ecosystem: ready-to-use crates

There's no need to reimplement the arena from scratch for every project. The Rust ecosystem offers several mature crates that also solve the limitations we mentioned (stale IDs, type-safety across different arenas):

- **[slotmap](https://crates.io/crates/slotmap)**: the most popular. Offers `SlotMap`, `DenseSlotMap`, and `HopSlotMap` with different trade-offs between iteration speed and insertion/removal speed. It uses a generational index internally to automatically invalidate stale IDs.

- **[thunderdome](https://crates.io/crates/thunderdome)**: an alternative to `slotmap`, optimized for simplicity and minimal ID size (packs generation and index into 64 bits).

- **[generational-arena](https://crates.io/crates/generational-arena)**: API similar to `slotmap`, focused on simplicity and correctness.

- **[id-arena](https://crates.io/crates/id-arena)**: arena with typed IDs but no generational index. Simpler, suited for append-only scenarios where removal isn't needed.

---

## When to use (and when to avoid) the pattern

The index-based arena is a powerful tool, but it's not the answer to everything.

**Use it when:**

- You have structures with bidirectional or cyclic references (graphs, trees, linked lists).
- Data has a uniform lifecycle (all created and destroyed in the same phase).
- You need simple serialization (indices are trivial to serialize).
- Performance matters: allocation is O(1), data is contiguous in memory.

**Avoid it when:**

- The structure is a simple tree with unidirectional ownership — `Box<T>` or `Vec<T>` are enough.
- You need compile-time guarantees on references — Rust's lifetimes, however complex, offer guarantees that indices cannot provide.
- Elements have very different lifecycles and removal is rare — a simple `Vec` with `Option<T>` might suffice.

---

## Conclusion

The doubly linked list in Rust is a problem that exposes a deep tension in the language: the ownership rules that make Rust safe are the same ones that make certain classic patterns hard to express. The arena with indices doesn't circumvent these rules — it respects them, moving all ownership to a single point and replacing pointers with numeric indices.

It's a trade-off: you lose some static guarantees (an index can become stale after a removal), but you gain simplicity, performance, and peace of mind. For cases where this risk is relevant, the ecosystem's crates offer ready-to-use solutions like generational indices and typed IDs — but that's material for a future article.

The next time the borrow checker blocks you on a structure with complex references, before reaching for `Rc<RefCell<T>>`, try asking yourself: _"What if I used an arena?"_.

> **Translation note:** This article was originally written in Italian and translated into English with the assistance of an AI language model. While every effort has been made to preserve the original meaning and technical accuracy, minor nuances may differ from the source text.
