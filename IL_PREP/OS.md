# 💻 OPERATING SYSTEMS
> **Status:** 🔴 Not started  
> **Context:** You've covered OSTEP (processes, scheduling, context switching, syscalls) — this file builds on that base

---

## Topic Checklist

### Processes & Threads
- [ ] Process vs thread — what they share, what's private
- [ ] Process creation (fork/exec model)
- [ ] Thread lifecycle — states, transitions
- [ ] Context switching — what gets saved, cost
- [ ] User threads vs kernel threads
- [ ] Concurrency vs parallelism — the distinction

### Scheduling (You've done this — just review)
- [ ] FIFO, SJF, STCF, Round Robin — trade-offs
- [ ] MLFQ — how it adapts to process behavior
- [ ] Lottery / stride scheduling
- [ ] CPU affinity, priority inversion

### Memory Management
- [ ] Virtual memory — why it exists, how it works
- [ ] Paging — page tables, TLB, page faults
- [ ] Segmentation
- [ ] Demand paging and lazy allocation
- [ ] Thrashing — what causes it, how to avoid
- [ ] Stack vs heap — who manages what
- [ ] Memory-mapped files

### Synchronization & Concurrency
- [ ] Race conditions — what causes them
- [ ] Mutex / lock — how it works under the hood
- [ ] Semaphore — binary vs counting
- [ ] Condition variables — wait/signal pattern
- [ ] Deadlock — four conditions (Coffman), detection, avoidance, prevention
- [ ] Spinlock vs sleep lock — when to use each
- [ ] Read-write locks

### File Systems
- [ ] Inodes — what they store
- [ ] Hard links vs soft links
- [ ] File system journaling — why crash recovery needs it
- [ ] Buffer cache / page cache

### I/O
- [ ] Blocking vs non-blocking I/O
- [ ] Async I/O — epoll, select, poll
- [ ] The event loop model (Node.js-style) and why it works
- [ ] DMA — direct memory access

### System Calls
- [ ] What happens during a syscall (mode switch, trap)
- [ ] Common syscalls: read, write, open, fork, exec, wait, mmap
- [ ] Cost of syscalls vs function calls

---

## Key Concepts to Articulate Clearly

### Process vs Thread
```
Process: Own address space, own file descriptors, own PID
Thread:  Shares address space + FDs with parent process, own stack + registers
```

### Deadlock — 4 Conditions (all must hold)
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

### Virtual Memory — Why It Matters
> Every process thinks it has the full address space. The OS + hardware (MMU) maps virtual → physical pages transparently. Key benefits: isolation, larger-than-RAM programs, copy-on-write.

---

## From Your Prior Work (OSTEP)
> You've covered: processes, FIFO/SJF/STCF/RR/MLFQ/lottery scheduling, context switching, syscalls. These are your strong zones — don't re-learn, just review.

---

## Notes
> Add here as you study

-
