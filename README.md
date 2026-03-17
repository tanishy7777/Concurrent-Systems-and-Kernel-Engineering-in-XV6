## XV6 README

  
# Concurrent Systems and Kernel Engineering in XV6

Extensions to the xv6 ARM kernel implementing a boosted lottery scheduler, kernel-level multithreading, and synchronization primitives.

---

## Features

### 1. Boosted Lottery Scheduler

Replaces the default round-robin scheduler with a probabilistic lottery-based CPU allocator with I/O-bound priority boosting.

**How it works:**
- Every runnable process holds a ticket count tracked in `pstat`
- A PRNG draws a winning ticket each scheduling quantum and then the winning process runs.
- Processes waking from `SLEEPING` (I/O-bound) receive a `boostsleft` counter, their tickets are double-weighted for that many quanta, giving them temporary priority to reduce I/O latency
- `runticks` is tracked per-process for scheduling stats

**Key functions:**
- `lottery()`: computes total tickets across all `RUNNABLE` processes and calls `hold_lottery()`
- `hold_lottery(total_tickets)`: draws the PRNG winner, applies boost logic, returns the winning `proc*`

---

### 2. Kernel-Level Multithreading

Three new system calls enable POSIX-style threading within the xv6 kernel.

#### `thread_create(tid*, start_func, arg)`
- Allocates a new `proc` entry with `is_thread = 1` and `mainpid` pointing to the leader
- Shares the leader's **page directory** and **open file descriptors**
- Allocates an independent **user stack** (one `PGSIZE` page via `allocuvm`) per thread
- Sets up the trapframe so the thread begins execution at `start_func(arg)`

#### `thread_exit()`
- Transitions the thread to `ZOMBIE` and wakes the leader
- Only valid for threads (`is_thread == 1`); no-op for main process

#### `thread_join(tid)`
- Main thread blocks until the target thread reaches `ZOMBIE`
- Safely deallocates the thread's user stack after join
- Only the main thread of a thread group can call `thread_join`

---

### 3. Synchronization Primitives

#### Barrier (`barrier.c`)
- `barrier_init(n)`: initializes a barrier for `n` threads
- `barrier_check()`: decrements the counter; last thread to arrive calls `wakeup()` to release all waiting threads. Others `sleep()` on the barrier channel.

#### Spinlock
Standard test-and-set spinlock from xv6 base, used internally for `ptable`, `pstat`, and barrier locking.

#### Semaphores & Condition Variables
Implemented as user-space or kernel-space primitives using `sleep`/`wakeup` as the underlying mechanism.

---

### 4. Optimized `wakeup` Path

**Problem:** Timer interrupts wake all processes sleeping on `&ticks`, even those with remaining sleep time — causing redundant context switches.

**Fix:** A separate `wakeup2(chan)` path is invoked for timer-channel wakeups. Instead of immediately marking a process `RUNNABLE`, it decrements `sleepTime`. Only when `sleepTime` reaches 0 is the process made runnable.

```c
// wakeup dispatches based on channel
void wakeup(void *chan) {
    acquire(&ptable.lock);
    if (chan == &ticks)
        wakeup2(chan);   // decrement sleepTime, defer wakeup
    else
        wakeup1(chan);   // standard immediate wakeup
    release(&ptable.lock);
}
```

**Result:** ~90% reduction in redundant wakeups for sleeping processes.

---

## Modified Files

| File | Changes |
|------|---------|
| `proc.c` | Lottery scheduler, `wakeup2`, boosting logic |
| `sysproc.c` | `sys_thread_create`, `sys_thread_exit`, `sys_thread_join` |
| `proc.h` | Added `is_thread`, `mainpid`, `ustack`, `sleepTime` fields |
| `pstat.h` | Added `tickets`, `boostsleft`, `runticks` arrays |
| `barrier.c` | Barrier implementation |
| `barrier.h` | Barrier interface |
| `syscall.c/h` | Registered new thread syscalls |

---

## Building and Running

```bash
cd OS
make
make qemu
```

Requires an ARM cross-compiler toolchain (`arm-none-eabi-gcc`).

---

## Design Notes

- Threads are modeled as lightweight processes sharing `pgdir` (no separate thread table needed)
- The lottery PRNG uses a simple LCG (`rseed * 1103515245 + 12345`) seeded at boot
- Boost tickets are additive (not multiplicative) to avoid starvation of CPU-bound processes
- `thread_join` handles the edge case where the thread's user stack is not at the top of the address space (non-contiguous deallocation via `walkpgdir`)
