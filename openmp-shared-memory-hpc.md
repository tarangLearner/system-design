# OpenMP & Shared-Memory Parallel Programming — Beginner Notes

> Source: HPC course, Week 4 – Faculty Session 6.
> Everything below is explained assuming you have never done parallel programming.

---

## 0. The big picture in one paragraph

Your CPU has multiple **cores** (independent little processors). Normal code uses only **one** core, so the other 7 (or 63) sit idle. **OpenMP** is a way to tell the compiler "hey, this chunk of code can be run by many cores at once" — by adding *one comment-like line* above your existing loop. You don't rewrite your program; you *annotate* it.

---

## 1. Shared-memory machines

**What it is:** One physical machine (one "node") that has several CPU cores, and all those cores plug into **the same RAM**.

**Analogy:** One kitchen, 8 cooks, **one shared fridge**. Any cook can open the fridge and grab/put anything, instantly, without asking anyone.

**Why it's nice:** If core 3 computes a value and stores it in a variable, core 5 can just read that variable. No sending messages, no copying data.

**Why it's dangerous:** Two cooks may reach for the same egg at the same time, or one overwrites what the other just put in. That's the whole "race condition" topic later in these notes.

> Contrast (coming later in the course): **distributed memory** = many separate machines, each with its own RAM. There, cores *must* send explicit messages (MPI). Harder, but scales to thousands of machines.

---

## 2. Thread, core, process — the vocabulary

| Term | Plain meaning |
|---|---|
| **Core** | Real physical hardware that executes instructions. You have a fixed number (e.g. 8). |
| **Thread** | A "worker" — an independent stream of instructions. Software concept. You can create as many as you want. |
| **Process** | Your running program. One process can contain many threads, and they **share memory**. |

The OS **schedules** threads onto cores. Threads ≠ cores. 16 threads on 8 cores is legal — they just take turns.

---

## 3. Why OpenMP instead of raw threads (pthreads)?

You *could* use `pthreads` (or `std::thread`) and manually create threads, split the work, join them back. OpenMP exists because:

1. **Code simplicity** — you keep your serial code and add `#pragma` lines. A 10,000-line serial program can be parallelised in a few hours.
2. **Incremental** — parallelise one loop, test it, then the next. No big rewrite.
3. **Portability** — any compiler that supports OpenMP will compile it. With pthreads you need that library present everywhere.
4. **Reversible** — remove the compiler flag and the pragmas become plain comments → you get your original serial program back. Great for debugging.

**The trade-off:** pthreads gives you finer control and *can* squeeze out more performance. OpenMP adds a small overhead.

**Faculty's argument:** You *could* also hand-write machine code to beat the compiler — but nobody does that for complex programs. Abstraction wins as complexity grows.

---

## 4. The Fork–Join model (the core idea of OpenMP)

```
   master thread  ──────►  (hits #pragma) ──┬──► thread 0 ──┐
                                            ├──► thread 1 ──┤
                                            ├──► thread 2 ──┼──► joined back ──► master continues
                                            └──► thread 3 ──┘
        serial              PARALLEL REGION                       serial
```

1. Program starts **serial**, with one thread (the **master thread**, always ID `0`).
2. It hits a `#pragma omp parallel`. → **Fork**: a *team* of threads is created.
3. All threads run the code inside `{ ... }`.
4. At the closing `}` → **Join**: threads finish, are destroyed, only the master continues.

Your only job as programmer: **mark which regions are safe to run in parallel.** The runtime does creation, work distribution, and destruction.

---

## 5. First program: Hello World

```c
#include <stdio.h>
#include <omp.h>          // needed for omp_* functions

int main(void) {
    #pragma omp parallel   // <-- everything in the block below runs on ALL threads
    {
        int id  = omp_get_thread_num();    // MY id: 0 .. n-1
        int nth = omp_get_num_threads();   // total number of workers
        printf("Hello, I am thread %d of %d\n", id, nth);
    }
    return 0;
}
```

**Line by line:**
- `#include <omp.h>` — without it, `omp_get_thread_num()` is undefined.
- `#pragma omp parallel` — the directive. The compiler creates the *machinery* to spawn threads.
- `omp_get_thread_num()` — "what's my worker badge number?" C-style numbering: `0` to `n-1`.
- `omp_get_num_threads()` — "how many co-workers are on this job?"

**Compile:**
```bash
gcc -fopenmp -O3 hello.c -o hello      # GCC / Clang
icc -qopenmp -O3 hello.c -o hello      # Intel compiler
```
> If you **omit** `-fopenmp`, the `#pragma` is silently ignored → you get a normal serial program. That is a feature, not a bug.

**Run:**
```bash
export OMP_NUM_THREADS=4
./hello
```

**Output (order changes every run!):**
```
Hello, I am thread 2 of 4
Hello, I am thread 0 of 4
Hello, I am thread 3 of 4
Hello, I am thread 1 of 4
```

### Key lessons from this tiny program
- **Output order is non-deterministic.** Thread 0 is not guaranteed to print first. Never assume ordering.
- **Thread 0 always exists** (minimum team size is 1), which is why it's called the master.
- Each thread runs *the same code* — the only difference is which **data** it touches.

---

## 6. Compile time vs Run time (a common confusion)

This confused a student in class, so worth spelling out:

| Stage | What happens |
|---|---|
| **Compile time** | The compiler sees `#pragma omp parallel for schedule(dynamic,16)` and generates machine code that *knows how* to spawn threads and *knows the policy* (dynamic, chunk 16). It does **not** create any thread. It doesn't even know how many threads there will be, or what the data will be. |
| **Run time** | The program actually executes, reads `OMP_NUM_THREADS`, creates that many threads, and hands out the actual data chunks according to the policy baked in at compile time. |

Think of it as: compile time writes the **rulebook**; runtime **plays the game**.

---

## 7. Controlling the number of threads

Three ways, in increasing priority:

```bash
# 1. Environment variable (outside the program)
export OMP_NUM_THREADS=8
```
```c
// 2. In code, globally
omp_set_num_threads(8);

// 3. Per region (highest priority)
#pragma omp parallel num_threads(4)
```

**If you set nothing:** most OSes default to the number of physical cores; some default to 1.

**Edge case asked in class:** `export OMP_NUM_THREADS=0` → the runtime warns *"value is too small, 1 will be used instead"* and runs with 1 thread. There's a safety net.

**Oversubscription:** 16 threads on 8 cores is *allowed*, but pointless for compute. 8 run, 8 wait, then swap. No speed-up, plus context-switching overhead.

> **Rule of thumb:** number of threads ≤ number of physical cores.

---

## 8. Work sharing — actually splitting the work

`#pragma omp parallel` alone makes every thread do **the same full job** 8 times. Useless. You need a **work-sharing construct**.

### 8.1 The parallel for loop (the workhorse)

```c
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i];
}
```

With `N = 10000` and 4 threads, the iteration range gets split into 4 pieces. Each thread does 2500 iterations. Done ~4× faster.

This single pragma does most of the heavy lifting in real HPC/ML code, because ML kernels are mostly big (often nested) loops.

### 8.2 THE critical rule: loop-carried dependence

OpenMP **does not check** whether your loop is safe to parallelise. It trusts you. If you're wrong, you get silently wrong answers.

**Safe (independent iterations):**
```c
c[i] = a[i] + b[i];       // iteration 251 doesn't care about iteration 343
```

**UNSAFE (loop-carried dependence):**
```c
a[i+1] = a[i] + b[i];     // needs a[i] which was produced by the PREVIOUS iteration
a[i]   = a[i-1] * 2;      // same problem
```
Here iteration 343 cannot start until 342 finished. Running these in parallel gives garbage.

> **Your job:** before adding `parallel for`, ask *"does iteration i read anything that iteration i-1 wrote?"* If yes → you must restructure the algorithm first.

---

## 9. Loop scheduling — how iterations are handed out

```c
#pragma omp parallel for schedule(<kind>, <chunk>)
```

### `static` (the default)
Split evenly, upfront. 10000 iterations / 4 threads → `0–2499`, `2500–4999`, `5000–7499`, `7500–9999`.

- **Pro:** lowest overhead. Decision made once.
- **Con:** if one core is busy (OS running your browser/Zoom), its chunk finishes late and *everyone waits*. Faculty's example: you could end up taking **2×** the time because one worker was stuck.

### `dynamic`
Break work into small **chunks**; whenever a thread finishes a chunk it grabs the next one from a queue.

- 10000 iterations in chunks of 250 → 40 chunks handed out on demand.
- If one core is unavailable, the other 3 just take more chunks. You get ≈1/3 of serial time instead of a stalled 1/2.
- **Pro:** automatic load balancing, great for **uneven work**.
- **Con:** overhead — someone has to manage handing out 40 pieces of work instead of 4.

### `guided`
Like dynamic, but chunk sizes **start large and shrink**. Big chunks first (low overhead), small chunks at the end (fine-grained balancing of the tail).

```c
#pragma omp parallel for schedule(guided, 16)   // 16 = minimum chunk size
```

**Analogy:** `static` = give each of 4 movers a quarter of the boxes upfront. `dynamic` = keep a pile and let movers grab 10 boxes at a time. `guided` = grab big armfuls first, small ones near the end so everyone finishes together.

| Kind | Overhead | Best when |
|---|---|---|
| static | lowest | every iteration costs the same |
| dynamic | higher | iterations have wildly different costs |
| guided | medium | uneven work, and you want a smooth finish |

---

## 10. Nested loops and `collapse`

ML code is full of 2D/3D loops:

```c
#pragma omp parallel for collapse(2)
for (int i = 0; i < M; i++)
    for (int j = 0; j < N; j++)
        C[i][j] = f(A[i][j], B[i][j]);
```

`collapse(2)` **merges the outer two loops into one big iteration space** of size `M*N`, then splits *that* across threads.

**Why you need it:** if `M = 3` but `N = 1,000,000`, parallelising only the outer loop gives you at most 3 threads of work. Collapsing gives 3,000,000 iterations to spread over all cores. Also tends to produce more **contiguous memory access**, which is cache-friendly.

---

## 11. Sections — functional parallelism

Instead of splitting *data*, split *different tasks*:

```c
#pragma omp parallel
{
    #pragma omp sections
    {
        #pragma omp section
        stage_A();          // one thread does this

        #pragma omp section
        stage_B();          // a different thread does this
    }
}
```

Use when you have genuinely different independent jobs (e.g., load data while computing something else), not N identical iterations. Good for irregular/recursive work.

---

## 12. SIMD / vectorisation — parallelism *inside* one core

This is a **different axis** of parallelism, and it's easy to mix up.

- **`omp parallel for`** = N *copies* of the instruction stream, each on a different core, each chewing one element at a time. (MIMD-ish)
- **SIMD** = **S**ingle **I**nstruction, **M**ultiple **D**ata = *one* instruction on *one* core processes **many numbers at once**, using special wide registers.

**Hardware:** Intel **AVX-512** = 512-bit-wide registers. That's 16 × 32-bit floats or 8 × 64-bit doubles processed by a single `add` instruction. ARM has **NEON/SVE** lanes.

```c
#pragma omp simd
for (int i = 0; i < N; i++)
    y[i] = a * x[i] + y[i];
```

**Analogy:** `parallel for` = hiring 8 cashiers. SIMD = giving each cashier a scanner that scans 16 items in one beep. You want **both**:

```c
#pragma omp parallel for simd
```

> `-O3` already tries to auto-vectorise. The pragma is you telling the compiler "trust me, it's safe."

---

## 13. Optimisation flags — free speed people forget

```bash
gcc -fopenmp -O3 program.c -o program
```

`-O3` alone (no parallelism at all) often gives **3–4× speed-up** because the compiler does auto-vectorisation, loop unrolling, inlining, etc. Always compile benchmarks with optimisation on. Comparing an `-O0` serial run against an `-O3` parallel run is a meaningless benchmark.

---

## 14. Data sharing: `shared` vs `private` (this is where bugs live)

```c
#pragma omp parallel for shared(a) private(tmp)
for (int i = 0; i < N; i++) {
    tmp  = expensive(a[i]);
    a[i] = tmp * 2;
}
```

| Clause | Meaning |
|---|---|
| **`shared`** | **One** copy of the variable, visible to all threads. **This is the default.** |
| **`private`** | **Each thread gets its own copy.** Uninitialised at entry, discarded at exit. |
| **`firstprivate`** | Private, but **initialised** with the value the variable had before the region. |
| **`lastprivate`** | Private, but the value from the **logically last iteration** is copied back out. |

### Why `private(tmp)` matters — the bug in slow motion

If `tmp` were shared with 4 threads:
1. Thread 0 computes `expensive(a[0])` and writes it to `tmp`.
2. Thread 1 computes `expensive(a[2500])` and writes it to the **same** `tmp`.
3. Thread 0 then reads `tmp` to do `a[0] = tmp*2` — but it now holds **thread 1's value**.

Result: garbage, and *different garbage on every run*. Making `tmp` private fixes it — each thread has its own scratchpad.

> **Rule:** every temporary/scratch variable written inside a parallel loop must be `private`. The loop index of a `parallel for` is automatically private.

**Student question answered:** *"Why not just declare `int tmp` inside the loop?"* — You can, and it's often cleaner (it becomes automatically private). You use `private()` when the variable must also exist *outside* the loop (before/after), so it can't be scoped inside.

---

## 15. Race conditions and reductions

### The classic race

```c
#pragma omp parallel for          // BUG
for (int i = 0; i < N; i++)
    sum += a[i];
```

`sum += a[i]` is not one operation, it's three: **read** `sum` → **add** → **write** `sum`.

| Time | Thread 0 | Thread 1 | `sum` in memory |
|---|---|---|---|
| t1 | reads 10 | | 10 |
| t2 | | reads 10 | 10 |
| t3 | computes 10+5=15 | | 10 |
| t4 | | computes 10+2=12 | 10 |
| t5 | writes 15 | | 15 |
| t6 | | writes 12 | **12** ← thread 0's work vanished |

Correct answer was 17. You got 12. And next run you might get 15, or 17. **Non-deterministic wrong answers** are the signature of a data race.

### Fix 1: `critical` — correct but defeats the purpose

```c
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    #pragma omp critical
    sum += a[i];               // only ONE thread inside at a time
}
```
Correct, but now every addition is serialised. You have 4 workers doing the work of 1, plus locking overhead. **Slower than the serial version.**

> `critical` is for the *rarest of rare* cases — protecting a multi-statement update. **Never** use it for a reduction like this.

### Fix 2: `atomic` — same idea, done in hardware

```c
#pragma omp atomic
sum += a[i];
```
Also serialises the update, but the CPU's own circuitry enforces it instead of a software lock → **cheaper than `critical`**. Limited to a **single simple update** (`+=`, `*=`, `++`, …) and needs hardware support.

**Analogy:** `critical` = your manager sits across the building and you must walk over for permission. `atomic` = the manager sits right next to you. Same rule, less walking.

### Fix 3: `reduction` — the RIGHT way

```c
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; i++)
    sum += a[i];
```

What it does under the hood:
1. Each thread gets a **private** copy of `sum`, initialised to the identity (0 for `+`, 1 for `*`).
2. Each thread accumulates its own **partial sum** at full speed, zero contention.
3. At the end of the region, the runtime combines the partial sums into the real `sum`.

**Analogy:** give 10 people 1000 numbers each; each adds their own pile; at the very end you add the 10 subtotals. Only the last tiny step is serial.

**Supported operators:** `+  -  *  &  |  ^  &&  ||  max  min`

> This same pattern (`reduce`) reappears in GPU programming and in MPI (`MPI_Allreduce`) across thousands of nodes. It's one of the most fundamental operations in all of parallel computing.

**Cost ranking — pick the cheapest that solves your problem:**

| Need | Use | Cost |
|---|---|---|
| Accumulate a value (sum/product/max/min) | `reduction` | lowest |
| One single variable update | `atomic` | low |
| Several statements must be updated together | `critical` | high |
| Separate two *phases* of the program | `barrier` | depends |

---

## 16. Synchronisation: `single`, `master`, `barrier`, `nowait`

### `single` vs `master`

```c
#pragma omp parallel
{
    #pragma omp single       // exactly ONE thread (any one) does this
    write_checkpoint();      // ...and there is an IMPLICIT BARRIER after it
    ...
}
```

```c
    #pragma omp master       // thread 0 specifically does this
    write_checkpoint();      // ...and there is NO barrier — others race ahead
```

| | Who runs it | Barrier after? |
|---|---|---|
| `single` | any one thread | **Yes** (implicit) |
| `master` | thread 0 only | **No** |

Typical use: printing logs, writing a file, one-time initialisation — things you don't want done 8 times.

### `barrier`

```c
#pragma omp parallel
{
    compute_partials();
    #pragma omp barrier      // EVERYONE waits here
    combine_results();       // nobody starts this until all partials are done
}
```

Without the barrier, thread 0 could start `combine_results()` while thread 3 is still computing → wrong answer.

**Implicit barriers already exist** at the end of `parallel for`, `sections`, and `single`. You rarely need an explicit one.

### `nowait` — removing an implicit barrier

```c
#pragma omp for nowait
for (...) { ... }
// threads that finish early immediately start the code below
```

**Use only when you are 100% sure** the following code doesn't depend on the loop being finished. Benefit: less idle time. Risk: subtle race bugs.

> More barriers = more red lights on the road. Correct, but slow. Use the minimum number that keeps you correct.

---

## 17. Locks — fine-grained control (last resort)

```c
omp_lock_t lock;
omp_init_lock(&lock);
...
omp_set_lock(&lock);      // acquire
// protected work
omp_unset_lock(&lock);    // release
...
omp_destroy_lock(&lock);
```

**Analogy:** one person enters a room, locks the door, does their thing, unlocks and leaves.

**Why locks when `critical` exists?** Because you can have **many independent locks**. With `critical`, protecting an array means only one thread can touch **the whole array** at a time. With locks, you can lock only element `7`, while other threads work freely on elements `4`, `5`, `6`.

**Danger:** more control = more ways to get it wrong → **deadlock** (thread A holds lock 1 waiting for lock 2, thread B holds lock 2 waiting for lock 1 → program freezes forever). Use only when reduction/atomic/critical genuinely can't express what you need.

---

## 18. Tasks — for irregular work

```c
#pragma omp parallel
{
    #pragma omp single
    {
        while (node) {
            #pragma omp task
            process(node);        // queued; any free thread picks it up
            node = node->next;
        }
        #pragma omp taskwait      // wait for all spawned tasks
    }
}
```

Loops need a known iteration count. Tasks don't. Use tasks for **linked lists, trees, graphs, recursion, variable-length sequences** — producer/consumer patterns where one thread generates work and a team consumes it.

---

## 19. Nested parallelism — handle with care

A thread is just a running piece of code, so nothing stops it from hitting another `#pragma omp parallel` and spawning *more* threads.

**The explosion:** 16 threads, each spawning 16 more = **256 threads** on a 64-core machine. Massive oversubscription, thrashing, slower than serial.

**Therefore: nested parallelism is OFF by default.** You must enable it deliberately (`omp_set_nested(1)` / `OMP_MAX_ACTIVE_LEVELS`) and cap the levels.

Related: **`threadprivate`** makes a global variable persistently private to each thread across parallel regions, so each thread behaves like its own independent serial program.

---

## 20. Thread affinity / pinning

By default the OS **migrates** threads between cores. It does this because it also has its own work (checking the network, keeping your SSH session alive, running your Zoom call). So thread 1 might move from core 1 → core 25 → core 6.

**Why that hurts:** when a thread moves, it loses its warm **cache** on the old core and has to refetch everything. On a big node this is a real slowdown.

**Pinning (affinity):** tell the OS "thread `k` stays on core `k`, no moving."

```bash
export OMP_PROC_BIND=close      # or spread / true
export OMP_PLACES=cores
```

**Practical tip from class:** on a 128-core node, launch 64 pinned threads and leave the remaining cores free for the OS. Then the OS never needs to evict your threads.

Only do this on **dedicated compute machines** — pinning on your laptop while browsing will backfire.

---

## 21. Heterogeneous cores (why "8 cores" isn't 8 equal cores)

The faculty's Mac M2 has 8 cores = **4 performance cores + 4 efficiency cores**.

**Analogy:** 4 fast horses and 4 slow horses. The slow ones eat far less food (power).

**Consequence:** give all 8 threads equal-sized chunks with `schedule(static)` and 4 of them finish much later → everyone waits at the barrier → **load imbalance**. This is exactly when `schedule(dynamic)` or `guided` pays for itself.

This mixed-core design is becoming the norm in new hardware, so don't assume all cores are equal.

---

## 22. False sharing (the invisible performance killer)

Memory doesn't move between RAM and cache one byte at a time — it moves in **cache lines** (typically 64 bytes).

If thread 0 writes `x[0]` and thread 1 writes `x[1]`, they're touching *different variables* — no logical race. But both live in the **same cache line**, so the hardware keeps invalidating and shuttling that line between the two cores' caches. Your code is correct but crawls.

**Fix:** pad the data, or give each thread its own well-separated accumulator (which is exactly what `reduction` does for you).

---

## 23. Debugging parallel correctness

**The golden workflow:**
1. Write and validate the **serial** version first. That's your reference answer.
2. Add pragmas incrementally.
3. Compare parallel output vs serial output after every change.
4. To go back to serial: just **drop `-fopenmp`**. Pragmas become comments.

**Vary the thread count:** run with 1, 2, 4, 8, 16, 64 threads. If the answer changes unpredictably → **you have a race condition.**

**But expect tiny differences in floating point!** Floating-point addition is **not associative**:
`(a + b) + c ≠ a + (b + c)` in finite precision. Change the thread count → change the summation order → change the round-off. Over billions of ML numbers this is visible in the least-significant bits.

So the test is:
- Same answer within a **sensible tolerance** across thread counts → fine.
- Wildly different / non-deterministic answers → **bug**.

This is not a logic error, it's how computers do arithmetic. Decide upfront whether you need bitwise reproducibility or tolerance-based comparison.

**Tools:** thread checkers such as **Intel Inspector** detect data races automatically and tell you *which variable* is under contention.

---

## 24. Quick reference cheat sheet

```c
#pragma omp parallel                     // fork a team of threads
#pragma omp parallel num_threads(4)      // ...with exactly 4
#pragma omp for                          // split loop iterations across the team
#pragma omp parallel for                 // both, combined
#pragma omp parallel for collapse(2)     // merge 2 nested loops first
#pragma omp for schedule(static)         // default: equal upfront chunks
#pragma omp for schedule(dynamic, 250)   // grab-a-chunk-when-free
#pragma omp for schedule(guided, 16)     // big chunks first, shrinking
#pragma omp for nowait                   // skip the implicit end barrier
#pragma omp simd                         // vectorise (SIMD within one core)
#pragma omp sections / section           // different tasks on different threads
#pragma omp single                       // one thread + implicit barrier
#pragma omp master                       // thread 0 only, no barrier
#pragma omp barrier                      // everyone waits here
#pragma omp critical                     // one thread at a time (software lock)
#pragma omp atomic                       // one thread at a time (hardware)
#pragma omp task / taskwait              // irregular / recursive work

// clauses
shared(x) private(t) firstprivate(t) lastprivate(t) reduction(+:sum)

// runtime API
omp_get_thread_num()  omp_get_num_threads()  omp_set_num_threads(n)
omp_init_lock() omp_set_lock() omp_unset_lock() omp_destroy_lock()

// environment
OMP_NUM_THREADS=8   OMP_PROC_BIND=close   OMP_PLACES=cores

// build
gcc -fopenmp -O3 prog.c -o prog
```

---

## 25. Mental checklist before parallelising any loop

1. Is this loop **hot** (does it dominate the runtime)? If not, don't bother.
2. Are the iterations **independent**? (Does iteration `i` read what `i-1` wrote?)
3. Which variables are **scratch**? → mark them `private`.
4. Am I **accumulating** anything? → use `reduction`, never `critical`.
5. Is the work per iteration **uniform**? → `static`. Uneven? → `dynamic`/`guided`.
6. Are the outer loops **too short**? → `collapse`.
7. Did I compile with **`-O3 -fopenmp`**?
8. Does the answer **match the serial version** across 1, 2, 4, 8 threads (within tolerance)?

---

## 26. Coming next in the course

- Parallelising real ML kernels (GEMM, convolutions, training steps) with OpenMP
- **GPU offload** via OpenMP `target` directives — a GPU is also a many-core machine, but organised differently (SIMT), so barriers/reductions behave differently
- Hybrid **OpenMP + MPI** for multi-node clusters
- Practical performance tuning and thread-debugging tools
