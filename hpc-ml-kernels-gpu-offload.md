# Parallelising ML Kernels, Memory Optimisation & GPU Offload — Beginner Notes

> Source: HPC course, Week 4 – Faculty Session 7 (follows on from [openmp-shared-memory-hpc.md](openmp-shared-memory-hpc.md)).
> Session 6 answered *"how do I run code on many cores?"*. This session answers *"why is my parallel code still slow, and how do I move it to a GPU?"*

---

## 0. The one-sentence summary

Adding more cores stops helping very quickly. **The real bottleneck is moving data**, so the whole game is: *rearrange your computation so data stays in fast memory and gets reused as much as possible* — first on the CPU (tiling), then on the GPU (shared memory + minimal host↔device transfers).

---

# PART A — Understanding what limits your speed

## 1. Arithmetic intensity (AI) — the single most important number

$$\text{Arithmetic Intensity} = \frac{\text{FLOPs performed}}{\text{Bytes moved}}$$

- **High AI** = lots of math per byte fetched → you are limited by how fast the CPU can *compute* → **compute-bound**.
- **Low AI** = very little math per byte fetched → the CPU sits idle waiting for data → **memory-bandwidth-bound**.

**Analogy:** you're a chef. AI = how much cooking you do per trip to the fridge. If you sprint to the fridge, grab one egg, crack it, and sprint back — you're not a slow cook, you're a slow *walker*. Adding 7 more chefs to the same fridge doesn't help; they queue at the fridge door.

> **This is why "just add more threads" stops working.** Once all cores are collectively saturating the memory bus, extra cores just wait.

## 2. The roofline model (quick recap)

Plot achievable GFLOP/s against arithmetic intensity:

```
 GFLOP/s
    ^
    |                    ______________  <- flat "roof": peak compute (hardware limit)
    |                   /
    |                  /   <- slanted part: memory bandwidth limit
    |                 /
    |________________/______________________> Arithmetic Intensity
             SAXPY, dot          GEMM (tiled)
```

- On the **slanted** part → you're bandwidth-limited. Optimise **memory access**.
- On the **flat** part → you're at hardware peak. You literally cannot go faster.

**Goal of all optimisation:** push your kernel to the right (raise AI) until it hits the flat roof.

## 3. The four kernel families in any ML workload

Every training step decomposes into these, and each has a *different* bottleneck:

| Kernel type | Examples | Arithmetic intensity | Bound by |
|---|---|---|---|
| **Element-wise** | activations (ReLU, GELU), add, scale | **very low** | memory bandwidth |
| **Reductions** | loss, norms, softmax denominator, mean/variance | **low** | memory bandwidth |
| **GEMM** (matrix×matrix) | linear layers, attention | **high** (in theory) | compute — *if* done right |
| **Convolutions** | vision models, edge detection, PDE solvers | high (usually reduced to GEMM) | compute |

Knowing which family a kernel belongs to tells you immediately whether optimising memory or optimising math will pay off.

---

# PART B — Three kernels, from naive to fast

## 4. SAXPY / AXPY — the "hello world" of numerical kernels

$$y_i = \alpha x_i + y_i$$

("**S**ingle-precision **A** times **X** **P**lus **Y**". Also DAXPY for double precision.)

```c
#pragma omp parallel for simd
for (int i = 0; i < n; i++)
    y[i] = alpha * x[i] + y[i];
```

Perfectly parallel — no dependencies at all. And yet:

**Count the arithmetic intensity (doubles = 8 bytes):**

| | |
|---|---|
| FLOPs per iteration | 1 multiply + 1 add = **2** |
| Bytes per iteration | read `x[i]` (8) + read `y[i]` (8) + write `y[i]` (8) = **24** |
| **Arithmetic intensity** | $2/24 = \mathbf{1/12}$ — terrible |

(`alpha` is a single constant that sits in a register/cache forever, so it doesn't count.)

**Live demo result (n ≈ 10⁸):** with 1 thread, 4 threads, 8 threads → **time barely changed**. The bandwidth was pinned at ~10 GB/s the whole time.

> **Lesson:** for a low-AI kernel, threads are useless. You are crying for bandwidth and no amount of workers fixes that.

## 5. Dot product — a parallel reduction

$$s = \sum_i x_i y_i$$

```c
double sum = 0.0;
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < n; i++)
    sum += x[i] * y[i];
```

`reduction(+:sum)` gives each thread a private partial sum and merges them at the end — no race condition, no serialisation. (See Session 6 notes.)

**The identical code pattern is used for:** vector norms, attention scores, similarity/cosine scores, loss functions.

### Live demo — and two very important lessons

| Threads | Time (ms) | GFLOP/s |
|---|---|---|
| 1 | 374 | 1.1 |
| 4 | 82 | 4.9 |
| 8 | ~68 | 5.5 |

**Anomaly spotted in class:** 374 / 82 = **4.5× speed-up from only 4× the workers**. That's *superlinear* — seemingly impossible.

**Then the faculty re-ran the 1-thread case:** 374 → 238 → 295 → 290 ms. And the 4-thread case: 82 → 93 → 83 ms.

**Resolution:** the 374 ms was an unlucky outlier in the tail of a distribution. The true mean is ~290 ms, so the real speed-up is ≈ 290/86 ≈ **3.4×** — perfectly normal.

> ### 🔴 The single most practical takeaway of the session
> **Never report a single timing number.** Run each configuration several times and report a mean (and ideally a spread). Timings vary because the OS is doing its own work, caches need warming, and threads get rescheduled. A one-shot number will lie to you.

*(Note: genuine superlinear speed-up **can** happen — when splitting data across threads makes each thread's working set small enough to fit in its private cache. But always rule out measurement noise first.)*

## 6. Naive GEMM — mathematically perfect, cache-hostile

$$C_{ij} = \sum_k A_{ik} B_{kj}$$

```c
#pragma omp parallel for collapse(2)
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++) {
        double s = 0.0;
        for (int k = 0; k < N; k++)
            s += A[i*N + k] * B[k*N + j];   // <-- B is accessed with stride N
        C[i*N + j] = s;
    }
```

**Why this is slow — the strided access problem.**

Memory is not fetched one number at a time. The hardware pulls a whole **cache line** (~64 bytes = 8 doubles) at once.

- `A[i*N + k]` walks along a **row** → consecutive addresses → one fetch gives you 8 useful values. 
- `B[k*N + j]` jumps `N` elements each step → **every single access pulls a whole cache line and uses exactly one number from it**, throwing away 7/8 of the bandwidth.

**Analogy:** the library delivers books one full shelf at a time. Reading `A` = you need every book on the shelf → perfect. Reading `B` = you need one book per shelf → they deliver 8 shelves, you read 8 books, and 56 books get carted away unused.

**Result:** GEMM has *theoretically* high arithmetic intensity, but the naive version achieves only a tiny fraction of peak FLOPs, because matrices are far too big to fit in cache and the same data gets evicted and re-fetched over and over.

## 7. Tiling / cache blocking — the fix

**Idea:** instead of computing one element of `C` completely (which requires a whole row of `A` and a whole column of `B`), compute **partial sums for a small square block of `C` at a time**, choosing the block size so that the three tiles (a tile of `A`, a tile of `B`, a tile of `C`) **all fit in L1/L2 cache simultaneously**.

```
   A                    B                    C
 ┌───┬───┬───┐       ┌───┬───┬───┐       ┌───┬───┬───┐
 │ a0│ a1│ a2│   ×   │ b0│   │   │   =   │ c0│   │   │
 ├───┼───┼───┤       ├───┼───┼───┤       ├───┼───┼───┤
 │   │   │   │       │ b1│   │   │       │   │   │   │
 ├───┼───┼───┤       ├───┼───┼───┤       ├───┼───┼───┤
 │   │   │   │       │ b2│   │   │       │   │   │   │
 └───┴───┴───┘       └───┴───┴───┘       └───┴───┴───┘

 c0 += a0×b0 ; then c0 += a1×b1 ; then c0 += a2×b2
 Each (a,b,c) trio is loaded once, fully reused, then discarded.
```

```c
#pragma omp parallel for collapse(2)
for (int ii = 0; ii < N; ii += T)             // T = tile size
  for (int jj = 0; jj < N; jj += T)
    for (int kk = 0; kk < N; kk += T)
      for (int i = ii; i < ii+T; i++)         // these 3 loops work
        for (int j = jj; j < jj+T; j++)       // entirely inside cache
          for (int k = kk; k < kk+T; k++)
            C[i*N+j] += A[i*N+k] * B[k*N+j];
```

**Crucially: the maths is identical.** You have not changed the formula for $C_{ij}$ at all — only the **order** in which the additions happen. You compute partial sums and accumulate, rather than finishing one element before starting the next.

**Effect:** each loaded byte is used many more times → **arithmetic intensity goes up** → you move right on the roofline and become compute-bound.

### Live demo results (N×N GEMM)

| Version | Threads | Time / rate | Speed-up |
|---|---|---|---|
| Naive, serial | 1 | 1.357 s | 1.0× (baseline) |
| `omp parallel for collapse(2)` | 4 | 0.445 s, 4.8 GFLOP/s | **3.0×** |
| Tiled + OpenMP | 4 | 5.6 GFLOP/s | ~3.5× |
| `omp collapse` | 8 | — | ~3.5× (barely better than 4 threads!) |
| **Tiled + OpenMP** | **8** | **8.1 GFLOP/s** | **~5.1×** |
| **Tiled, single thread** | **1** | — | **1.3×** |

### Two conclusions worth memorising

1. **Going from 4 → 8 threads gave almost nothing for the untiled version** (3.0× → 3.5×). It was already bandwidth-starved. Tiling is what let extra threads actually pay off (5.1×).
2. **Tiling alone, on ONE thread, with zero parallelism, gave 1.3× speed-up.**

> **Optimise memory access *before* you optimise parallelism.** If your memory access pattern is bad, throwing cores at it is just burning more resources on a problem you should have fixed first.

---

# PART C — Other common parallel patterns

## 8. Tree reduction — $O(\log n)$ instead of $O(n)$

Summing $n$ numbers one at a time takes $n$ sequential steps. Pair them up instead:

```
 3   1   4   2   7   5   0   6      (8 values)
  \ /     \ /     \ /     \ /
   4       6      12       6        step 1  (4 additions in parallel)
    \     /         \     /
      10              18            step 2  (2 additions in parallel)
        \            /
             28                     step 3  (1 addition)
```

$\log_2 8 = 3$ steps instead of 8. This is exactly what `reduction(+:sum)` does internally, and it's the same structure used for gradient averaging across workers and for GPU/MPI `allreduce`.

## 9. Scan / prefix sum — the "impossible" one

Output where each element is the running total: `[3,1,4,2] → [3,4,8,10]`.

```c
a[i+1] = a[i+1] + a[i];    // classic loop-carried dependence!
```

Looks strictly sequential — element `i+1` needs element `i`. **But parallel scan algorithms exist** (Hillis–Steele, Blelloch) that do it in $O(\log n)$ steps. 

Shows up in: sorting, sequence models, stream compaction, sparse matrix construction. To be covered later in the course.

> **Lesson:** "it has a loop-carried dependence" doesn't always mean "it can't be parallelised" — it means "the naive form can't be; find a different algorithm."

## 10. Histogram — privatise, then reduce

**Problem:** bin a billion values into 10 buckets. Every thread wants to do `bins[b]++` on the same small array → massive contention and data races.

**Wrong fix:** `#pragma omp atomic` on every increment → serialises everything.

**Right fix — per-thread private histograms, then reduce:**

```c
#pragma omp parallel
{
    int local[NBINS] = {0};                 // each thread's own bins
    #pragma omp for nowait
    for (long i = 0; i < n; i++)
        local[bin_of(data[i])]++;           // zero contention

    for (int b = 0; b < NBINS; b++) {       // merge at the end
        #pragma omp atomic
        bins[b] += local[b];
    }
}
```

This is the **same "privatise then reduce" pattern** as `reduction(+:sum)`, just applied to an array. Contention happens once per thread instead of once per element.

## 11. Stencil operations — the heart of convolutions and PDEs

A **stencil** computes each output from a small neighbourhood of inputs:

```
          [ i-1, j ]
[ i, j-1 ] [ i, j ] [ i, j+1 ]      out[i][j] = f(these 5 values)
          [ i+1, j ]
```

**Where you meet them:**
- **Edge detection** — how fast is the grey level changing between neighbouring pixels? A sharp gradient = an edge.
- **Blurring / smoothing** — the inverse: replace each pixel with an average of its neighbours.
- **PDE solvers** — heat transfer, fluid mechanics, electromagnetics, and even the **Black–Scholes** option-pricing equation. Finite-difference methods are all stencils.

**The cost:** a more accurate gradient needs a wider neighbourhood (more "rings" of neighbours). More neighbours = more data per output = another **tiling problem**. You load a tile plus a **halo** (the border ring it needs from adjacent tiles) into cache and reuse it.

## 12. im2col — turning convolution into GEMM

Convolutions are awkward to optimise directly. The standard trick: **unfold each image patch into a column** of a big matrix, then the entire convolution becomes one large matrix–matrix multiply.

Why bother? Because GEMM is the *most optimised operation in all of computing*. Reducing your problem to GEMM means you inherit decades of tuning for free.

> **General strategy:** whatever your ML kernel is, try to express it as a GEMM, then call a tuned library.

---

# PART D — Never hand-roll: use tuned libraries

## 13. The library landscape

| Platform | Libraries |
|---|---|
| **CPU** | BLAS, LAPACK, OpenBLAS, Intel MKL |
| **NVIDIA GPU** | cuBLAS, cuDNN, CUTLASS |
| **Sparse** | cuSPARSE, PETSc, Trilinos |

**A humbling history lesson:** BLAS and LAPACK were written in **Fortran, in the 1970s**. That's a ~50-year legacy still running underneath every modern AI framework. We think of AI as 5–10 years old; the mathematical bricks it's built on are half a century old. *We are standing on the shoulders of giants.*

**Tensor cores:** modern GPUs have dedicated circuitry that performs small matrix–matrix multiplies in hardware. Yet another reason to route your maths through a library that knows how to use them.

### The hidden cost: these libraries trade memory for speed

Tuned GEMM implementations keep **multiple redundant copies of tiles** in re-packed, cache-friendly layouts. So an optimised implementation may use **3–4× more memory** than a naive one. That's a deliberate trade — memory for bandwidth efficiency.

### Choosing the right function matters as much as using a library

Every BLAS-style library exposes dozens of routines (`axpy`, `dot`, `gemm`, `gemv`, inverse, factorisations…). Picking a *mathematically correct but structurally wrong* routine kills performance.

**Clearest example:** using a **dense** GEMM routine to multiply **sparse** matrices. Correct answer, catastrophic speed.

> **You have almost certainly already used BLAS without knowing it** — NumPy, PyTorch and TensorFlow all call into it underneath.

## 14. Sparse matrices

A **sparse** matrix is one where most entries are zero.

**The social-graph example from class:** take 500 million users on a social platform and build a "who is connected to whom" matrix. That's a 500M × 500M matrix, but the average person has maybe 100–500 connections. So **essentially every entry is zero**.

Two separate problems:
1. **Storage waste** — storing $2.5\times10^{17}$ mostly-zero entries is impossible.
2. **Wasted compute** — multiplying by zero and adding zero is pure waste.
3. **Poor locality** — the few non-zeros are scattered unpredictably (your friends aren't stored next to you in memory), so caching barely helps.

**Sparse storage formats** store only the non-zeros plus their coordinates:
- **CSR** (Compressed Sparse Row) — the most common.
- **COO** (Coordinate list).

**When to actually use sparse:**

| Fraction non-zero | Use sparse? |
|---|---|
| ~5% | **Yes**, clearly |
| ~60% | **No** — the indexing overhead outweighs the savings |

Sparse kernels have *lower raw efficiency* than dense ones — but they're solving a far smaller problem. **You must measure your actual sparsity to decide.** (Same social graph restricted to one school-year cohort becomes dense again — everyone knows everyone.)

**Iterative solvers:** for $Ax = b$ with sparse $A$, you never compute $A^{-1}$. Iterative methods (via PETSc, Trilinos) are the right tool.

---

# PART E — Kernel fusion and data layout

## 15. Kernel fusion — stop re-reading the same array

**Softmax**, done the obvious way, is three separate passes over the array `x`:

$$\text{softmax}(x)_i = \frac{e^{x_i - \max_j x_j}}{\sum_j e^{x_j - \max_j x_j}}$$

1. Pass 1: find $\max_j x_j$ (a max-reduction)
2. Pass 2: compute $\sum_j e^{x_j - \max}$ (a sum-reduction)
3. Pass 3: divide each element (element-wise)

If `x` is a billion elements, it does not fit in cache. So each pass streams the *entire* array from RAM and throws it away — **three full trips through memory** for a handful of FLOPs. Hopelessly memory-bound.

**Fusion:** merge the passes so each chunk of `x` is loaded once and all the work on it is done while it's still in cache. This is precisely the idea behind **FlashAttention** and similar optimisations.

**Same story for normalisation layers** (BatchNorm / LayerNorm): compute mean (reduction) → compute variance (reduction) → scale & shift (element-wise). Fuse them.

> **Rule:** whenever a kernel is memory-bound, ask *"what else can I do to this data while it's already in cache?"*

## 16. Data layout: Array-of-Structs vs Struct-of-Arrays

This is a **design decision you must make up front** — changing it later means rewriting everything downstream.

Say each patient has temperature, blood pressure, RBC count.

### Array of Structs (AoS)
```c
struct Patient { double temp, bp, rbc; };
struct Patient patients[N];      // temp,bp,rbc | temp,bp,rbc | temp,bp,rbc | ...
```
One patient's fields are adjacent in memory.

### Struct of Arrays (SoA)
```c
struct Patients {
    double temp[N];              // temp,temp,temp,... 
    double bp[N];                // bp,bp,bp,...
    double rbc[N];               // rbc,rbc,rbc,...
};
```
The same field across all patients is adjacent.

### Which one?

| Your workload | Best layout | Why |
|---|---|---|
| **Record lookup** — "show me patient #2032" | **AoS** | that patient's 3 fields arrive in one cache line |
| **Analytics / ML** — "correlate temperature across all N patients" | **SoA** | all temperatures are contiguous → **coalesced access**, and SIMD/GPU can load 8–16 at once |

With AoS in an ML workload, reading every patient's temperature means **striding** through memory — exactly the naive-GEMM disease from §6.

> **For ML, statistics, SIMD and GPUs: use Struct-of-Arrays.** For database-style record retrieval: Array-of-Structs.

---

# PART F — Moving to the GPU

## 17. Why a GPU at all?

ML kernels are **massively data-parallel** — the same simple operation applied to millions of elements independently. That is exactly what a GPU is built for.

**The pizza analogy (from the architecture lecture):** a CPU is a few sports cars — very fast, great at complicated routes. A GPU is a thousand scooters — each slow, but if you have 1000 identical pizzas to deliver to 1000 nearby addresses, the scooters win overwhelmingly.

A single GPU offers far more FLOP/s **and** far more bandwidth (~1 TB/s between GPU cores and VRAM) than a CPU socket.

## 18. The two hard constraints

### (a) Fixed, small memory
Top cards have ~80–96 GB of VRAM, and **you cannot upgrade it**. Unlike a workstation where you add RAM sticks, whatever the GPU shipped with is what you have forever.

**Consequence:** even *inference* on a 120-billion-parameter model doesn't fit on one GPU. Multi-GPU is mandatory for any serious modern workload (and that opens up a whole new topic: multi-GPU programming, covered later).

### (b) The CPU↔GPU transfer is the bottleneck
Every program starts on the CPU. Data on disk or in the cloud must be read **into CPU memory first**, then shipped to the GPU in batches. The GPU cannot read a file or draw to your screen — results must come back to the CPU.

```
  Disk / Cloud  ──►  CPU RAM  ──►  GPU VRAM  ──►  compute  ──►  back to CPU RAM
                              ▲                              ▲
                              └──── SLOWEST LINK IN THE CHAIN ┘
```

> **This transfer will kill your performance if you're careless.** Minimising it is the #1 GPU optimisation.

## 19. Two ways to program a GPU

| Approach | Tools | Effort | Control | Portability |
|---|---|---|---|---|
| **Directive-based offload** | OpenMP `target`, OpenACC | minimal code change | less | vendor-neutral |
| **Explicit kernels** | CUDA, SYCL | significant rewrite | maximum | CUDA = NVIDIA only |

Same trade-off as OpenMP vs pthreads from Session 6. **Horses for courses:** simple kernel or "I want results today"? Use directives. Squeezing every last percent out of a critical kernel? Write it explicitly.

## 20. OpenMP GPU offload

```c
#pragma omp target teams distribute parallel for \
        map(to: a[0:n], b[0:n]) map(from: c[0:n])
for (int i = 0; i < n; i++)
    c[i] = a[i] + b[i];
```

Decoding it clause by clause:

| Clause | Meaning |
|---|---|
| **`target`** | "compile this region for the GPU and run it there." The compiler detects the GPU architecture (NVIDIA/AMD/Intel) and generates its machine code — a GPU has a completely different instruction set from the CPU. |
| **`teams`** | create multiple **teams** (→ thread blocks). |
| **`distribute`** | spread loop iterations *across* the teams. |
| **`parallel for`** | spread iterations *within* each team across its threads. |
| **`map(to: ...)`** | copy this data **CPU → GPU** before the region. |
| **`map(from: ...)`** | copy this data **GPU → CPU** after the region. |

Leaving out `teams distribute` details lets the compiler pick what's optimal for the detected card — usually the right choice.

### ⚠️ Why `map` is not optional

If you omit the `map` clauses, OpenMP **plays it safe and copies everything both ways**:

- Before: copies `a`, `b`, **and `c`** to the GPU — but `c` is uninitialised garbage! Pure waste.
- After: copies `a`, `b`, and `c` back — but `a` and `b` were never modified! Pure waste again.

That's **2× the traffic on the slowest link in the system.** Being explicit about what goes in and what comes out is one of the highest-leverage things you can do.

## 21. GPU execution hierarchy (essential vocabulary)

```
GRID  ──►  BLOCKS  ──►  THREADS  ──►  WARPS
(whole     (groups that   (individual   (groups of 32 threads that
 launch)    share fast     workers)      physically execute together,
            memory +                     in lockstep)
            can sync)
```

- You launch **thousands** of threads on a GPU (vs ~8 on a CPU).
- Threads are grouped into **blocks** (commonly 256 or 512 threads/block). Threads in a block share a small, very fast **shared memory** and can synchronise with each other.
- Blocks are internally scheduled in **warps** of **32** threads. Even a 512-thread block executes as 16 warps, not all at once.
- Block size and warp size are **hardware-determined**, not arbitrary choices.

## 22. The same kernel in CUDA

```c
__global__ void add(const float* a, const float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;   // this thread's global index
    if (i < n)                                        // bounds guard — see below
        c[i] = a[i] + b[i];
}

// launch:
add<<<(n + 255) / 256, 256>>>(d_a, d_b, d_c, n);
```

**Decoding it:**

- **`__global__`** — "this function runs on the GPU, is called from the CPU." Tells the compiler to emit GPU machine code.
- **`blockIdx.x * blockDim.x + threadIdx.x`** — *the* fundamental CUDA idiom. "Which block am I in?" × "how big is a block?" + "which thread am I inside my block?" = my unique global index. This appears in essentially every CUDA kernel.
- **`<<< blocks, threadsPerBlock >>>`** — the "chevron" launch syntax, unique to CUDA.
- **`(n + 255) / 256`** — **ceiling division**. Integer division truncates, so `n/256` would under-allocate blocks whenever `n` isn't a multiple of 256. Adding 255 first rounds up.
  - `n = 1000` → `(1000+255)/256 = 4` blocks = 1024 threads, covering all 1000 elements.
  - `n = 1` → `(1+255)/256 = 1` block = 256 threads; 255 of them do nothing.
- **`if (i < n)`** — the bounds guard. Those extra padding threads must not write past the end of the array. **Forgetting this is a classic memory-corruption bug.**

### The explicit memory management CUDA forces on you

```c
cudaMalloc(&d_a, n*sizeof(float));                             // allocate on GPU
cudaMemcpy(d_a, h_a, n*sizeof(float), cudaMemcpyHostToDevice); // CPU -> GPU
add<<<blocks, 256>>>(d_a, d_b, d_c, n);                        // run kernel
cudaMemcpy(h_c, d_c, n*sizeof(float), cudaMemcpyDeviceToHost); // GPU -> CPU
cudaFree(d_a);                                                  // free
```

**Host** = CPU side. **Device** = GPU side. Every array exists **twice** — `h_a` on the host, `d_a` on the device — and *you* keep them in sync.

> This is exactly what the OpenMP `map` clause does for you invisibly. CUDA just makes you write it out.

## 23. GPU shared-memory tiling

The same tiling idea from §7, one level down:

```c
__shared__ float tileA[T][T];
__shared__ float tileB[T][T];

// every thread loads one element of the tiles
tileA[ty][tx] = A[...];
tileB[ty][tx] = B[...];

__syncthreads();          // WAIT until all threads in the block finished loading

// now compute using the tiles — they're in ultra-fast shared memory
```

**`__syncthreads()`** is a barrier *within a block* — it guarantees every thread sees the fully-loaded tile before anyone starts computing. Without it you read garbage.

> **CPU cache blocking and GPU shared-memory tiling are the same idea:** get a block of data into the fastest memory available and reuse it as many times as possible before letting it go.

## 24. CUDA from Python — Numba

```python
from numba import cuda

@cuda.jit
def add(a, b, c, n):
    i = cuda.grid(1)          # equivalent to blockIdx*blockDim + threadIdx
    if i < n:
        c[i] = a[i] + b[i]

blocks = (n + 255) // 256
add[blocks, 256](d_a, d_b, d_c, n)     # square brackets instead of chevrons
```

`@cuda.jit` triggers **just-in-time compilation**: the first time the function is called, Numba compiles it to GPU machine code. Easiest on-ramp to CUDA if you already know Python.

## 25. OpenACC — the cousin

```c
#pragma acc parallel loop copyin(a[0:n], b[0:n]) copyout(c[0:n])
for (int i = 0; i < n; i++)
    c[i] = a[i] + b[i];
```

**Why it exists:** GPUs appeared in the early 2000s; CUDA arrived ~2007–08. Around 2012 people wanted a *directive-based* way to use GPUs, but the OpenMP committee was slow to act — so a separate group created OpenACC. OpenMP later added `target` offload, so today the two are competing cousins with near-identical concepts (`copyin`/`copyout` ≈ `map(to:)`/`map(from:)`).

## 26. Portability frameworks

**CUDA locks you to NVIDIA.** If your code must survive a hardware change, consider:

| Framework | What it is |
|---|---|
| **OpenCL** | "Open Compute Language" — low-level, CUDA-equivalent, works across NVIDIA/AMD/Intel/FPGAs. Callable from many languages. |
| **SYCL** | Modern C++ layer, single-source, cross-vendor. |
| **Kokkos** | Performance-portability library (think STL-for-HPC): write once, compile for CPU, NVIDIA, AMD, etc. |
| **OpenMP `target` / OpenACC** | Directive-based, also vendor-neutral. |

> Portability is a first-class concern in HPC. Hardware outlives no code, but code outlives a lot of hardware.

---

## 27. Consolidated decision guide

```
Is my kernel slow?
│
├─ Measure arithmetic intensity (FLOPs / bytes)
│
├─ LOW AI (element-wise, reductions) ──► memory-bound
│   ├─ Fuse adjacent kernels (softmax, layernorm)
│   ├─ Fix data layout (AoS → SoA)
│   ├─ Ensure contiguous / coalesced access
│   └─ Accept it: more threads will NOT help
│
├─ HIGH AI (GEMM, conv) ──► should be compute-bound
│   ├─ Am I actually hitting peak? If not:
│   ├─ Tile / cache-block to raise effective AI
│   ├─ collapse() nested loops for enough parallelism
│   └─ Better still: call cuBLAS / MKL / OpenBLAS
│
├─ Is my matrix mostly zeros (>~90%)? ──► sparse format (CSR) + sparse library
│
└─ Still not enough? ──► GPU offload
    ├─ Simple / quick   ──► OpenMP target or OpenACC
    ├─ Maximum control  ──► CUDA / SYCL
    ├─ ALWAYS minimise CPU↔GPU transfers (explicit map/copy clauses)
    └─ Tile into __shared__ memory + __syncthreads()
```

---

## 28. Cheat sheet

```c
/* ---------- CPU: memory optimisation ---------- */
#pragma omp parallel for simd                   // vectorised parallel loop
#pragma omp parallel for reduction(+:sum)       // safe accumulation
#pragma omp parallel for collapse(2)            // merge nested loops
// tiling: loop over blocks of T, then inside each block

/* ---------- GPU: OpenMP offload ---------- */
#pragma omp target teams distribute parallel for \
        map(to: a[0:n], b[0:n]) map(from: c[0:n])

/* ---------- GPU: OpenACC ---------- */
#pragma acc parallel loop copyin(a[0:n]) copyout(c[0:n])

/* ---------- GPU: CUDA ---------- */
__global__ void k(...) { int i = blockIdx.x*blockDim.x + threadIdx.x; if (i<n) {...} }
k<<<(n+255)/256, 256>>>(d_a, d_b, d_c, n);
cudaMalloc / cudaMemcpy / cudaFree
__shared__  +  __syncthreads()

/* ---------- GPU: Numba (Python) ---------- */
@cuda.jit
def k(a, b, c, n):
    i = cuda.grid(1)
k[blocks, threads](d_a, d_b, d_c, n)
```

**Libraries:** BLAS · LAPACK · OpenBLAS · Intel MKL (CPU) — cuBLAS · cuDNN · CUTLASS (GPU) — cuSPARSE · PETSc · Trilinos (sparse)

---

## 29. Ten things to remember from this session

1. **Arithmetic intensity** (FLOPs ÷ bytes) decides whether you are compute-bound or memory-bound.
2. **SAXPY-type kernels are hopeless** — AI of 1/12, threads don't help, bandwidth is the ceiling.
3. **Naive GEMM is cache-hostile** because of strided access to `B`.
4. **Tiling changes only the order of operations**, not the maths — but it transforms performance.
5. **Tiling alone, single-threaded, gave 1.3×.** Fix memory before adding cores.
6. **Never report one timing number** — always average over multiple runs.
7. **Privatise, then reduce** — for sums, histograms, and anything accumulated.
8. **Fuse memory-bound kernels** (softmax, layernorm) so data is touched once.
9. **SoA for ML/GPU, AoS for record lookup** — and decide before you write the code.
10. **On the GPU, the CPU↔GPU transfer is the enemy.** Always say explicitly what goes in and what comes out.

---

## 30. Coming next in the course

- Tool demos: measuring roofline, identifying memory- vs compute-bound kernels, thread debugging
- Detailed walkthroughs of the OpenMP and CUDA sample codes (to be shared by faculty)
- Multi-GPU and multi-node programming (MPI, mpi4py), collective operations
- Advanced GPU programming and performance tuning

> **Course admin note:** no quizzes or midterms — **100% of the grade is a final project presentation.** Topics form to be circulated; groups or individual both allowed; past-project examples will be shared.
