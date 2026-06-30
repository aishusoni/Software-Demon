# ⚙️ C/C++ MASTERY — Root Tracker
> **Goal:** Deep, complete understanding of C/C++ — memory model, language internals,
> performance, concurrency — to the level required for HFT-grade systems engineering
> **Status:** 🟡 Starting
> **Context:** Currently SDE2 at Honeywell (Python/Django), considering HFT path,
> learning C/C++ as a fundamentally different paradigm from daily work

---

## 📁 File Index

| File | What it tracks |
|------|----------------|
| [C_FUNDAMENTALS.md](./C_FUNDAMENTALS.md) | Pure C — memory, pointers, manual management |
| [CPP_CORE.md](./CPP_CORE.md) | C++ specific — OOP, RAII, templates, STL |
| [MEMORY_MODEL.md](./MEMORY_MODEL.md) | Stack/heap, pointers, memory layout, undefined behavior |
| [CONCURRENCY_CPP.md](./CONCURRENCY_CPP.md) | Threads, atomics, memory ordering, lock-free structures |
| [PERFORMANCE.md](./PERFORMANCE.md) | Cache lines, SIMD, compiler optimizations, profiling |
| [HFT_SPECIFIC.md](./HFT_SPECIFIC.md) | Low-latency patterns, what HFT firms actually care about |
| [PROJECTS.md](./PROJECTS.md) | Hands-on projects log — build real things, not just read |

---

## 📺 Primary Resource — Mike Shah's "The C++ Programming Language" (247 videos)
> https://www.youtube.com/playlist?list=PLvv0ScY6vfd8j-tlhYVPYgiIyXduu6m-L
> Recommended by a friend. Confirmed deep, well-regarded, low-level focused —
> Mike Shah is a systems/graphics engineer who explicitly teaches toward
> HFT/game-dev/performance-critical C++. Good fit for this goal.

**Confirmed structure (topic blocks, not exact video order):**

```
Block 1 — Language Basics
  Setup/compilation, types, operators, control flow, functions, arrays

Block 2 — Memory Fundamentals
  Stack vs heap vs static memory (confirmed dedicated video on this)
  Pointers, references, the 'static' keyword

Block 3 — Classes (32 parts — the spine of the course, go slow here)
  5  Avoiding copies (delete, copy constructor, pass by reference)
  6  Operator overloading
  7  Member initializer lists
  8  Structs in C++
  9  RAII (Resource Acquisition Is Initialization)
  10 Rule of Five
  11 friend functions (and why to avoid them)
  12 Explicit constructors, list initialization
  13-15 Inheritance (intro, access levels, constructor chaining)
  16 Virtual functions (dynamic dispatch)
  17 Virtual destructors (why base class destructor must be virtual)
  18 Understanding the vtable (popular interview question)
  19 Interfaces (pure virtual functions)
  20 Multiple inheritance (with caution)
  26 Value/zero initialization
  27 In-class initializers
  28 Delegating constructors
  29 Class data layout (optimizing for size)
  30 pIMPL (pointer to implementation)
  31 The 'this' keyword
  32 static member variables/functions

Block 4 — Generics/Templates
  1 Templates introduction
  2 Template functions (abbreviated function templates)
  3 Multiple template parameters, non-type parameters
  4 Full/partial specialization
  5 Variadic templates

Block 5 — STL
  Containers, iterators, algorithms

Block 6 — Move Semantics & Modern C++
  rvalue references, move constructors, smart pointers, C++11-23 features

Block 7 — Concurrency
  threads, atomics, memory ordering

Companion courses by same instructor (separate playlists):
  GDB debugger course — learn this alongside, debugging C is non-negotiable
  VIM course — optional, workflow speed
```

**How to use this alongside the phase plan below:** the phases below are the
CONCEPTS to master. This playlist is the RESOURCE to learn them from. Watch
in topic blocks, but always pause and do the "explain it back + write code"
loop with me — don't just passively watch 247 videos.

---

## 🗺️ Learning Path (Roughly Sequential)

### Phase 1 — C Fundamentals (Foundation)
- [ ] Memory model: stack vs heap, how a process's memory is laid out
- [ ] Pointers — fully, not just "address of a variable"
- [ ] Pointer arithmetic, arrays vs pointers, the relationship between them
- [ ] Manual memory management — malloc/free, common bugs (leaks, double-free, use-after-free)
- [ ] Structs, unions, bit fields
- [ ] Function pointers
- [ ] The C preprocessor — macros, conditional compilation
- [ ] Compilation pipeline — preprocessor → compiler → assembler → linker

### Phase 2 — C++ Core
- [ ] RAII (Resource Acquisition Is Initialization) — the central C++ idiom
- [ ] Constructors/destructors, copy vs move semantics
- [ ] References vs pointers
- [ ] Smart pointers — unique_ptr, shared_ptr, weak_ptr (and why they exist)
- [ ] Templates — generic programming, template metaprogramming basics
- [ ] STL — vector, map, unordered_map internals, iterators
- [ ] Virtual functions, vtables, polymorphism under the hood
- [ ] Exception handling — how it actually works, cost implications

### Phase 3 — Memory Deep Dive
- [ ] Stack frames, calling conventions
- [ ] Heap allocators — how malloc actually works internally
- [ ] Memory alignment, padding, struct layout
- [ ] Undefined behavior — what it actually means, why it's dangerous
- [ ] Valgrind, AddressSanitizer — tools to catch memory bugs
- [ ] Custom allocators — why and how

### Phase 4 — Concurrency
- [ ] std::thread, std::mutex, std::condition_variable
- [ ] Atomics — std::atomic, compare-and-swap
- [ ] Memory ordering — relaxed, acquire-release, sequential consistency
- [ ] Lock-free data structures — what they are, why they're hard
- [ ] False sharing, cache line contention

### Phase 5 — Performance Engineering
- [ ] CPU cache hierarchy — L1/L2/L3, cache lines, cache misses
- [ ] Branch prediction
- [ ] SIMD — vectorized instructions
- [ ] Compiler optimization flags, what -O2/-O3 actually do
- [ ] Profiling tools — perf, gprof

### Phase 6 — HFT-Specific Patterns
- [ ] Why HFT avoids heap allocation in the hot path
- [ ] Lock-free queues for inter-thread communication
- [ ] Kernel bypass networking concepts (DPDK, etc.)
- [ ] Latency measurement — how HFT firms actually measure microseconds
- [ ] What HFT interviews actually test

---

## 🎯 Approach

Same method as system design prep:
1. First principles explanation — why the concept exists, what problem it solves
2. Mental model — a clean analogy
3. You explain it back in your own words
4. Hands-on — write actual code, not just read about it
5. Connect to real systems (Redis is written in C, understand why certain choices were made)

---

## 📌 Session Log

| Session | Topic | What clicked | What to revisit |
|---------|-------|--------------|------------------|
| — | — | — | — |

---

## 🔗 Connection to Existing Knowledge

You already understand systems thinking deeply (system design prep). This C/C++ track
is about going BELOW the abstractions you already reason about well:

```
System design = how do components talk to each other, what are the trade-offs
C/C++ mastery = what is actually happening in memory and CPU when any of
                this code executes

Example: You know Redis is single-threaded + epoll for speed.
         C/C++ mastery = understanding what epoll() actually does at the
         syscall level, why Redis's data structures are cache-friendly,
         how its memory allocator (jemalloc) works
```
