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
