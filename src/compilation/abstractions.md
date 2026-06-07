# Zero-Cost Abstractions

"Zero-cost abstraction" is a C++ slogan: you should be able to use high-level abstractions (classes, templates, lambdas) without paying a runtime penalty compared to hand-written low-level code. This is true *when you use the right abstractions correctly*. But many common abstractions are not zero-cost, and knowing the difference is critical for performance work.

## What Makes an Abstraction Zero-Cost?

An abstraction is zero-cost if:
1. It compiles to the same machine code you would write by hand.
2. It imposes no runtime overhead beyond what a hand-rolled equivalent would require.

Examples: `std::vector` is zero-cost compared to a manually managed dynamic array. `std::sort` is zero-cost (and usually better) compared to a hand-written quicksort. Template-based `std::accumulate` is zero-cost compared to a raw loop.

Non-zero-cost abstractions: `std::function` (type erasure has overhead), `std::shared_ptr` (reference counting), virtual functions (indirect call), `iostream` (heavy formatting machinery).

## Virtual Functions

```rust
trait Compute {
    fn compute(&self) -> i32;
}

struct Derived;

impl Compute for Derived {
    fn compute(&self) -> i32 { 42 }
}

// Dynamic dispatch (vtable lookup + indirect call, like C++ virtual):
fn call_virtual(obj: &dyn Compute) -> i32 {
    obj.compute()  // Indirect call through vtable
}

// Static dispatch (zero-cost alternative — monomorphized, fully inlined):
fn call_static<T: Compute>(obj: &T) -> i32 {
    obj.compute()  // Direct call after monomorphization
}
```

Cost: indirect branch (vtable lookup + indirect call), no inlining across the call. If `call_virtual` is in a hot loop and the concrete type varies, the indirect branch predictor may mispredict.

Alternatives:
- **Templates + CRTP** (Curiously Recurring Template Pattern): static polymorphism, fully inlined.
- **`std::variant` + `std::visit`**: tagged union, the visitor is essentially a switch on the type index.
- **Manual type tagging**: an `enum` + a `switch`. Crude but fast and predictable.

## `std::function`

```rust
// Heap-allocated closure (like std::function — may allocate):
let func: Box<dyn Fn(i32) -> i32> = Box::new(|x| x * 2);
let result = func(5);  // Indirect call — may involve heap allocation

// Zero-cost alternatives:
// 1. Generic parameter (fully inlined, zero cost):
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }

// 2. Raw function pointer (direct call if known at compile time):
let func: fn(i32) -> i32 = |x| x * 2;
let result = func(5);
```

Cost: `std::function` uses type erasure. For small callables (up to ~32 bytes depending on implementation), the object is stored inline (no heap allocation). For larger callables, a heap allocation occurs. Every call goes through an indirect function pointer.

Alternatives:
- **Template parameter**: `template<typename F> void apply(F func) { func(x); }` — fully inlined, zero cost.
- **`std::move_only_function`** (C++23): like `std::function` but non-copyable, enabling better small-object optimization.
- **Raw function pointer**: `int (*func)(int) = ...` — direct call if the function is known at compile time; indirect call otherwise.

## Pointer-Chasing Containers

`std::list`, `std::map`, `std::set`, `std::unordered_map` are all node-based containers. Each element is a separately allocated node with pointers to neighbors. Iterating over them chases pointers through memory, incurring a cache miss at every node (or every few nodes, if the allocator placed them nearby by chance).

```rust
// Rust: LinkedList is node-based, same cache issues as std::list.
use std::collections::LinkedList;

let lst: LinkedList<i32> = LinkedList::from([1, 2, 3, 4, 5]);
let mut sum = 0;
for x in &lst {  // Cache miss on every element
    sum += x;
}
```

The `std::vector` equivalent does a sequential scan through contiguous memory:

```rust
// Rust: Vec is contiguous, like std::vector.
let vec: Vec<i32> = vec![1, 2, 3, 4, 5];
let mut sum = 0;
for &x in &vec {  // Cache hits after the first element
    sum += x;
}
```

For iteration-heavy workloads, `std::vector` can be 10–100× faster than `std::list`, even when `std::list` theoretically has better algorithmic complexity for the insertion pattern. Cache locality dominates.

**Prefer `std::vector` (or `std::array`) unless you need reference stability or O(1) mid-sequence insertion/deletion.** When you do need node-based structures, the `data-structures/` chapter covers cache-optimized alternatives.

## Bounds-Checked Access

`std::vector::at(i)` throws on out-of-bounds access. `std::vector::operator[](i)` does not. The performance difference is real: `at()` includes a bounds check and a conditional branch.

```rust
// With bounds check (branch in debug, may be elided in release):
for i in 0..n {
    sum += vec[i];  // Potential bounds check
}

// Without bounds check (unsafe, no branch):
for i in 0..n {
    sum += unsafe { vec.get_unchecked(i) };  // No bounds check
}

// Even better (iterator — no bounds check, no indexing):
for &x in &vec {  // No bounds check, no indexing
    sum += x;
}
```

Use `operator[]` and range-based for loops in hot code. Reserve `at()` for debugging or when bounds checking is genuinely needed (untrusted input).

## `std::min` and Branchless Code

```rust
let smaller = std::cmp::min(a, b);
```

`std::min` typically generates a conditional move (`cmov`), which is branchless and fast. But this depends on the compiler and the context. If `std::min` isn't generating `cmov`, write:

```rust
let smaller = if a < b { a } else { b };  // Check assembly — should be cmov
```

## Designing Abstraction-Friendly Interfaces

The key to zero-cost abstractions:

1. **Use templates, not runtime polymorphism**. The compiler can inline through templates; it can't through virtual calls.
2. **Pass function objects by value or template parameter, not `std::function`**.
3. **Use contiguous containers (`std::vector`, `std::array`) as the default.** Reach for node-based containers only with evidence.
4. **Return by value** — move semantics and RVO/NRVO (named return value optimization) make returning large objects efficient.
5. **Document your performance contracts.** If a function is designed for the hot path, document that callers should pass a template functor, not `std::function`.

A well-designed template library compiles to the same assembly as a hand-rolled C loop. The abstraction exists only in the source code; it evaporates during compilation. This is the promise of C++, and when it works, it's beautiful. The skill is recognizing when it's not working and knowing what to use instead.
