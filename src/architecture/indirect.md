# Indirect Branching

Most branches go to a fixed address known at compile time. **Indirect branches** go to an address computed at runtime. They are essential for polymorphism, function pointers, and large switch statements — and they introduce a performance cost that direct branches don't have.

## How Indirect Jumps Work

A direct jump: `jmp .L123` — the target address is encoded in the instruction.
An indirect jump: `jmp rax` — the target is in a register, loaded from memory, or computed.

Indirect calls work the same way: `call rax` instead of `call func`.

The CPU's branch predictor handles indirect branches with a dedicated structure called the **indirect branch predictor** (or branch target buffer, BTB). For a direct branch, the predictor only needs to guess *taken/not taken*. For an indirect branch, it needs to guess the *target address* — a much harder problem.

When the predictor gets the target wrong, the penalty is the same as a mispredicted direct branch (~15–20 cycles), plus the cost of computing or loading the correct target.

## Virtual Function Dispatch

The most common source of indirect branches in C++:

```cpp
class Base {
public:
    virtual void process() = 0;
};

class DerivedA : public Base {
public:
    void process() override { /* A's implementation */ }
};

class DerivedB : public Base {
public:
    void process() override { /* B's implementation */ }
};

void call_process(Base *obj) {
    obj->process();  // indirect call through vtable
}
```

The call `obj->process()` compiles to something like:

```asm
mov rax, [rdi]          ; load vtable pointer from object
call [rax + offset]     ; call through vtable entry
```

The vtable lookup involves two memory accesses:
1. Load the vtable pointer from the object (`[rdi]`).
2. Load the function pointer from the vtable (`[rax + offset]`).

Both of these are data-dependent — the second load can't start until the first completes. This chain of dependent loads adds latency before the call target is even known.

If `call_process` is called with a mix of `DerivedA` and `DerivedB` objects, the indirect branch predictor must guess which target each time. If one type dominates (>90%), prediction is accurate and the cost is low. If types are evenly mixed, prediction is a coin flip and you pay the full misprediction penalty.

## Devirtualization

The compiler can eliminate virtual dispatch when it can prove the concrete type:

```cpp
DerivedA obj;
obj.process();  // compiler knows this is DerivedA::process()
```

Or with whole-program optimization / LTO:

```cpp
Base *obj = new DerivedA();
obj->process();  // with LTO and escape analysis, the compiler might devirtualize this
```

Use `final` on classes and methods to enable devirtualization:

```cpp
class DerivedA final : public Base { ... };
// Compiler now knows any Base* pointing to DerivedA must call DerivedA's methods
```

For hot code where virtual dispatch cannot be devirtualized, consider alternative polymorphism mechanisms: `std::variant` + `std::visit`, templates + CRTP, or manual type tagging with a `switch`.

## Switch Statements

A `switch` with many cases compiles to one of several strategies.

**Sequential if-else chain** (few cases, sparse values):
```c
switch (x) {
    case 1: return 10;
    case 5: return 50;
    case 100: return 1000;
    default: return 0;
}
```
Compiles to: `cmp x, 1; je ...; cmp x, 5; je ...; cmp x, 100; je ...`

**Jump table** (many cases, dense values):
```c
switch (x) {
    case 0: return 10;
    case 1: return 20;
    case 2: return 30;
    // ... 100 cases ...
}
```
Compiles to:
```asm
cmp eax, 100
ja .Ldefault             ; bounds check
movsxd rax, [.LJMPTABLE + rax*4]  ; load target offset
add rax, .LJMPTABLE      ; compute absolute address
jmp rax                  ; indirect jump!
```

The jump table is an array of code addresses in read-only memory. The compiler bounds-checks the input, then uses it as an index into the table. This is O(1) regardless of the number of cases — but it requires an indirect jump.

**Binary search** (medium number of moderately-spaced cases): the compiler may emit a binary search tree of `cmp` + `jcc` instructions.

**Hybrid**: For ranges, the compiler might use multiple jump tables or combine sequential checks with table lookups.

## Performance Implications

1. **Keep hot polymorphism predictable**: If a virtual function is called millions of times with the same concrete type, the indirect branch predictor learns it. If it alternates between different types, you pay the misprediction penalty.

2. **Dense switch values enable jump tables**: Use sequential integer tags rather than arbitrary values for hot `switch` statements.

3. **Small switches may be faster as if-else chains**: The indirect jump through a table costs more than a few predicted direct branches.

4. **Function pointers are indirect calls**: Passing callbacks through function pointers has the same cost as virtual dispatch. Templates + lambdas allow the compiler to inline through the call and eliminate the indirection entirely.

5. **`std::function` adds another layer of indirection**: It uses type erasure with an internal virtual call, even for small functors. For hot code, prefer template parameters or `std::move_only_function` (C++23) with guaranteed small-object optimization.

## Measuring Indirect Branch Cost

You can see indirect branch mispredictions with:

```bash
perf stat -e branch-misses,branch-instructions ./your_program
```

A `branch-misses` rate above 2–3% on indirect branches suggests the predictor is struggling. The fix is usually to reduce polymorphism in hot paths or to sort data so that calls with the same target are grouped together.
