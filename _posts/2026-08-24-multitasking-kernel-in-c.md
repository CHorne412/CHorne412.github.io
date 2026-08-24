---
title: "What I learned writing a multitasking kernel in C"
date: 2026-08-24
tags: [c, systems, operating-systems]
description: >-
  Context switching, priority queues, and a first-fit allocator — the pieces
  that have to agree with each other before a process can be paused and resumed
  without noticing.
---

For a systems course last spring, four of us built a multitasking operating
system in C for 32-bit x86 — process management, a dispatcher, and a heap
allocator, with nothing underneath doing the hard parts for us.

The code lives in a private university repository, so this is the design rather
than the source. My work was concentrated in the process control blocks, the
command handler, the serial driver, and the memory allocator.

## A context is a struct that has to match the assembly

Multitasking means stopping a process mid-instruction, remembering enough to
resume it, running something else, and coming back — with the paused process
unable to tell it was ever interrupted. On x86 that "enough" is concrete: the
segment registers, the general-purpose registers, the instruction pointer, and
the flags.

We defined that as a packed struct — `ss`, `gs`, `fs`, `es`, `ds`, then `edi`,
`esi`, `ebp`, `esp`, `ebx`, `edx`, `ecx`, `eax`, then `eip`, `cs`, `eflags`.
Sixty bytes, and **the field order is not a style choice**. The interrupt
service routine pushes registers onto the stack in a specific order, and the
struct is just a typed window onto that memory. Reorder a field in the header
and the kernel doesn't fail to compile — it reads `eflags` where `cs` lives and
jumps somewhere insane.

That was my first real lesson in this project: in C at this level, the compiler
checks almost nothing you care about. The struct and the assembly agree because
you made them agree, and the only feedback if you're wrong is a machine that
triple-faults.

## The PCB, and five queues

Each process gets a Process Control Block holding its name, class, priority,
execution state, dispatching state, a stack pointer, and a 1 KB stack embedded
directly in the struct. Each PCB also carries `next` and `prev` pointers,
because every PCB is always a member of exactly one doubly-linked queue:

- **ready** — runnable, waiting for the CPU
- **blocked** — waiting on something
- **suspended ready** and **suspended blocked** — the same two states, but
  administratively suspended
- **running** — the one with the CPU

Insertion into the ready queue is priority-ordered: walk from the head until
you find the first process whose priority is worse than the one being inserted,
and splice in ahead of it. Lower number means higher priority. Because the list
is kept in priority order on insert, the dispatcher never has to search — the
next process to run is always the head.

That's a deliberate trade. Insertion is O(n), dispatch is O(1), and dispatch
happens far more often.

## The dispatcher is cooperative, and that shows

Scheduling runs through a system call handler. A process yields by issuing
`sys_req(IDLE)`; the handler saves the outgoing context pointer into the
process's PCB, moves it back to the ready queue, takes the head of the ready
queue, marks it running, and returns *its* saved context pointer. The interrupt
return then restores that context, and a different process resumes as if
nothing happened.

The thing worth being honest about: this is **cooperative**, not preemptive. A
process that never calls `sys_req` never gives up the CPU, and nothing in the
kernel can take it away. A timer interrupt driving preemption is the obvious
next step, and it's the first thing I'd add.

## First-fit, splitting, and coalescing

The heap starts as one 50,000-byte block and is tracked by Memory Control
Blocks in two address-ordered doubly-linked lists: one free, one allocated.

Allocation walks the free list, takes the first block big enough, and — if that
block is larger than requested — splits it, leaving the remainder as a new free
block. Freeing moves the block back to the free list and then merges adjacent
free blocks so the heap doesn't shred itself into unusable fragments over time.

Keeping both lists sorted by address is what makes coalescing possible at all:
"is this block adjacent to the next one" is only a cheap question if the list
is in address order. That constraint isn't obvious until you try to write the
merge without it.

One honest note: the code comments call this best-fit in places, and there's a
`best_fit` variable, but it breaks out of the loop on the first block that fits
— so it is first-fit. Naming that survived a refactor is its own small lesson.

## The bug: allocating into the wrong place entirely

The allocator worked, then stopped working, in the way memory bugs do — not
with a crash at the guilty line, but with data quietly landing where it
shouldn't.

Saved blocks weren't being fitted into the right location. The allocator would
skip the next available gap and keep the pointer parked at the previous block
instead of walking the free list to find the first space that fit. Freed memory
was never really recycled; the heap just marched forward until it ran out.

There were two causes underneath that one symptom.

The first was pointer arithmetic doing something other than what it reads like.
Splitting a block computed the leftover's address roughly like this:

```c
MCB *new_block = (MCB *)best_fit->start_addr + size;
```

That looks like "advance `size` bytes." It isn't. Adding to a typed pointer
advances in units of the pointed-to type, so this jumped `size * sizeof(MCB)`
bytes — off by a factor of the struct's size, landing the new free block far
outside where it belonged. Casting to `char *` first makes the arithmetic
byte-wise and the intent explicit:

```c
char *new_block_addr = (char *)best_fit->start_addr + size;
```

The second was subtler and cost more time: the split was reading `current`,
the loop cursor, where it should have read `best_fit`, the block actually
chosen. When the first block examined happened to be the one selected, those
two are identical and everything works. They diverge only once the allocator
has to walk past a block to find a fit — which is exactly the case that was
broken, and exactly the case that only shows up after enough allocation and
freeing to fragment the heap.

The fix landed the next day: free blocks now go back into the free list in
address-sorted order, and merging runs immediately after. Without that
ordering, "is the next block adjacent to this one" is unanswerable, so
neighbouring gaps never coalesced and the walk kept missing space that was
right there.

How I found it: serial-port print statements, in quantity. There's no debugger
attached to a kernel booting under QEMU, so the loop was print the pointer,
boot, read the number, decide it's absurd, move the print somewhere earlier,
boot again. The moment the fix became obvious was seeing an address roughly
thirty-two times further along than it should have been — `sizeof(MCB)` is
sixteen bytes, and a number that wrong doesn't come from logic, it comes from
units.

**What I took from it:** in C, a type doesn't just describe what's in memory,
it silently redefines what `+ 1` means. And when a bug only appears after the
heap fragments, the reproduction case is the thing to build first — I spent
longer trying to trigger it by hand than the fix itself took.

## What I'd do differently

**Preemption.** Cooperative scheduling was enough to pass, but it means a
runaway process wedges the machine.

**Fewer special cases in the coalescing logic.** The merge routine accumulated
guards for edge cases we hit during testing, and by the end it was doing
arithmetic on block sizes that took real effort to follow. That's usually a
sign the data structure wants rethinking rather than more conditionals.
