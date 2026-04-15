# Lab: Parallel Programming with OpenMPI

---

## Table of Contents

1. [What is OpenMPI and How Does it Work?](#1-what-is-openmpi-and-how-does-it-work)
2. [Installing OpenMPI on Ubuntu 24.04](#2-installing-openmpi-on-ubuntu-2404)
3. [Rank, Synchronization, and Critical Sections](#3-rank-synchronization-and-critical-sections)
4. [Classic SPMD: Parallel Dot Product](#4-classic-spmd-parallel-dot-product)
5. [Pipeline Parallelism](#5-pipeline-parallelism)
6. [Master-Worker Pattern](#6-master-worker-pattern)
7. [Exercise 1: Amdahl's Law Table](#7-exercise-1-amdahls-law-table)
8. [Exercise 2: Process Scheduler Simulation](#8-exercise-2-process-scheduler-simulation)

---

## 1. What is OpenMPI and How Does it Work?

### The Problem OpenMPI Solves

Modern CPUs have multiple cores, and compute clusters have multiple machines — yet a standard C++ program uses exactly one core on one machine. To exploit the rest, you need a framework that can **spawn, coordinate, and communicate between** multiple independent processes. OpenMPI is the most widely used implementation of the **MPI standard** (Message Passing Interface) that does exactly this.

### Architecture Overview

OpenMPI operates on the **message passing** model. There is no shared memory between processes — each process has its own private address space. Instead of reading a shared variable, processes must **explicitly send and receive messages** to exchange data.

```
┌──────────────────────────────────────────────────────────┐
│                    mpirun -np 4 ./prog                   │
└──────────────────────────────────────────────────────────┘
         │           │           │           │
    ┌────▼───┐  ┌────▼───┐  ┌────▼───┐  ┌────▼───┐
    │ Rank 0 │  │ Rank 1 │  │ Rank 2 │  │ Rank 3 │
    │ Core 0 │  │ Core 1 │  │ Core 2 │  │ Core 3 │
    │ own RAM│  │ own RAM│  │ own RAM│  │ own RAM│
    └────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘
         └───────────┴───────────┴───────────┘
                   MPI Communication Layer
              (shared memory / network / InfiniBand)
```

Each box is a **full OS process** — not a thread. They run independently and communicate only when they call MPI functions.

### Key Concepts

**Communicator:** A group of processes that can talk to each other. The default communicator `MPI_COMM_WORLD` contains all processes launched by `mpirun`. You can create sub-groups for more complex topologies.

**Rank:** A unique integer ID assigned to each process within a communicator, starting from 0. This is how processes identify themselves and address messages to each other. Think of it as a process's "name" in MPI's world.

**Point-to-Point Communication:** Direct message exchange between two specific ranks.
- `MPI_Send(...)` — blocking send
- `MPI_Recv(...)` — blocking receive
- `MPI_Isend / MPI_Irecv` — non-blocking variants

**Collective Communication:** Operations involving all processes in a communicator simultaneously.
- `MPI_Bcast` — one process sends data to all others
- `MPI_Scatter` — one process splits data and sends chunks to all
- `MPI_Gather` — all processes send data to one
- `MPI_Reduce` — all processes contribute to a single computed result (sum, max, etc.)
- `MPI_Barrier` — all processes wait until every process reaches this call

**The Execution Model (SPMD):** Every process runs the **same binary**, but takes different code paths determined by its rank. This is called **Single Program, Multiple Data (SPMD)** — the program text is singular, but each instance operates on different data and can follow different logic branches.

### How MPI Bootstraps

When you run `mpirun -np 4 ./prog`, the MPI runtime:
1. Forks 4 copies of `./prog` (or distributes across nodes in a cluster)
2. Assigns each a unique rank within `MPI_COMM_WORLD`
3. Sets up communication channels (shared memory for same-node, TCP/InfiniBand for multi-node)
4. Each process calls `MPI_Init(&argc, &argv)` to connect to the runtime
5. Processes run concurrently until `MPI_Finalize()` — after which no MPI calls are valid

### MPI vs Threads (pthreads / OpenMP)

| Feature | OpenMPI | Threads (OpenMP/pthreads) |
|---|---|---|
| Memory model | Distributed (no sharing) | Shared memory |
| Data exchange | Explicit MPI calls | Read/write shared variables |
| Scalability | Across many nodes/machines | Single machine only |
| Synchronization | MPI_Barrier, blocking sends | Mutexes, barriers, atomics |
| Overhead | Higher per-message | Lower (cache-friendly) |
| Race conditions | Not possible (no shared memory) | Must be managed carefully |
| Best for | Large-scale, distributed HPC | Fine-grained parallelism on one machine |

> In practice, **hybrid programming** (MPI between nodes + OpenMP within a node) is common in HPC. OpenMPI supports this natively via `MPI_THREAD_MULTIPLE`.

---

## 2. Installing OpenMPI on Ubuntu 24.04

### Step 1 — Update package lists

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install OpenMPI

```bash
sudo apt install -y openmpi-bin openmpi-common libopenmpi-dev
```

This installs:
- `openmpi-bin` — the `mpirun` / `mpiexec` launcher
- `libopenmpi-dev` — headers and libraries for compiling MPI programs
- `openmpi-common` — shared configuration files

### Step 3 — Verify installation

```bash
mpirun --version
mpic++ --version
```

Expected output:
```
mpirun (Open MPI) 4.1.x
...
g++ (Ubuntu 13.x) ...
```

### Step 4 — Check available slots (logical cores)

```bash
nproc
```

This tells you the maximum number of processes you can run in parallel on this machine without oversubscription.

### Step 5 — Run a Hello World to confirm everything works

Create `hello_mpi.cpp`:

```cpp
#include <mpi.h>
#include <iostream>

int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    std::cout << "Hello from rank " << rank
              << " of " << size << " processes." << std::endl;

    MPI_Finalize();
    return 0;
}
```

Compile and run:

```bash
mpic++ hello_mpi.cpp -o hello_mpi
mpirun -np 4 ./hello_mpi
```

Expected output (order may vary — processes run concurrently):
```
Hello from rank 2 of 4 processes.
Hello from rank 0 of 4 processes.
Hello from rank 3 of 4 processes.
Hello from rank 1 of 4 processes.
```

> The non-deterministic order confirms these are truly independent concurrent processes. ✅

---

## 3. Rank, Synchronization, and Critical Sections

### Rank

Every process has a rank — an integer from `0` to `size - 1`. Rank 0 is conventionally the **master** or **root** process. It is used to:
- Control program flow (`if (rank == 0) { ... }`)
- Address messages (`MPI_Send(..., dest=2, ...)`)
- Perform root operations in collectives (`MPI_Reduce(..., root=0, ...)`)

```cpp
int rank, size;
MPI_Comm_rank(MPI_COMM_WORLD, &rank);  // "Who am I?"
MPI_Comm_size(MPI_COMM_WORLD, &size);  // "How many of us are there?"
```

### Synchronization — MPI_Barrier

`MPI_Barrier(MPI_COMM_WORLD)` is a **global synchronization point**. No process can pass it until **all** processes in the communicator have reached it.

```
Rank 0: ──────────────────────────┤BARRIER├──────────►
Rank 1: ──────────────────┤BARRIER├──────────────────►
Rank 2: ──────────────────────────────────┤BARRIER├──►
                                           ↑
                                  All three arrive here
                                  then all proceed together
```

Use `MPI_Barrier` when you need to ensure all ranks have completed a phase (e.g., all finished writing partial results) before any rank proceeds to the next phase (e.g., gathering results).

```cpp
// All ranks do some work...
do_work(rank);

MPI_Barrier(MPI_COMM_WORLD);  // Wait for everyone

// Now safe to proceed knowing all work is done
if (rank == 0) collect_results();
```

> ⚠️ **Deadlock warning:** Every process must call `MPI_Barrier` — if one process skips it (e.g., inside an `if` branch), the program will hang forever waiting for that process.

### Critical Sections in MPI

Unlike threads, MPI processes have **no shared memory**, so there are **no race conditions** in the traditional sense. However, you can still have **logical critical sections** — operations that must be performed by only one process, or in a specific order.

**Pattern 1 — Only rank 0 does I/O:**

```cpp
// Printing from all ranks simultaneously produces garbled output
// Instead, serialize output through rank 0:
for (int r = 0; r < size; ++r) {
    if (rank == r) {
        std::cout << "Rank " << rank << " reporting: " << my_value << std::endl;
    }
    MPI_Barrier(MPI_COMM_WORLD);  // force sequential printing
}
```

**Pattern 2 — Token passing (explicit ordering):**

```cpp
if (rank == 0) {
    do_critical_work();
    int token = 1;
    MPI_Send(&token, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);  // pass baton to rank 1
} else {
    int token;
    MPI_Recv(&token, 1, MPI_INT, rank-1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    do_critical_work();
    if (rank < size - 1)
        MPI_Send(&token, 1, MPI_INT, rank+1, 0, MPI_COMM_WORLD);
}
```

**Pattern 3 — MPI_Reduce for safe aggregation:**

Instead of all processes writing to a shared sum (which would race in threads), MPI uses `MPI_Reduce` — each process contributes its local value and MPI handles the safe aggregation:

```cpp
double local_sum = compute_my_part(rank);
double global_sum = 0.0;
MPI_Reduce(&local_sum, &global_sum, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
// global_sum is valid only on rank 0
```

---

## 4. Classic SPMD: Parallel Dot Product

### What is SPMD?

**Single Program, Multiple Data (SPMD)** is the most common MPI pattern:
- All ranks execute the same program
- Each rank receives a **different chunk of data**
- Each rank computes a **partial result** on its chunk
- Results are **reduced** (combined) to produce the final answer

The dot product `A · B = Σ A[i] * B[i]` is a textbook SPMD example: each rank computes a partial sum over its slice of the vectors, and then all partial sums are reduced to a global sum on rank 0.

```
Vector A: [a0, a1, a2, a3 | a4, a5, a6, a7 | a8, a9, a10, a11 | a12,a13,a14,a15]
                ↓                  ↓                   ↓                  ↓
           Rank 0              Rank 1              Rank 2              Rank 3
        partial_sum0        partial_sum1        partial_sum2        partial_sum3
                ↓                  ↓                   ↓                  ↓
                └──────────────── MPI_Reduce (SUM) ────────────────────┘
                                        ↓
                                  global_dot (rank 0)
```

### The Code

```cpp
#include <mpi.h>
#include <iostream>
#include <vector>
#include <random>
#include <numeric>
#include <cmath>
#include <chrono>

int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);   // this process's ID
    MPI_Comm_size(MPI_COMM_WORLD, &size);   // total number of processes

    const int N = 1 << 22;  // ~4 million elements
    std::vector<float> h_a, h_b;

    // ── Only rank 0 generates the full vectors ────────────────────────────────
    // In SPMD, one process (conventionally rank 0) owns the data and distributes it.
    // Generating on all ranks would give different random sequences.
    double serial_dot = 0.0;
    auto start_serial = std::chrono::high_resolution_clock::now();

    if (rank == 0) {
        h_a.resize(N);
        h_b.resize(N);
        std::mt19937 gen(42);
        std::uniform_real_distribution<float> dist(0.0f, 1.0f);
        for (int i = 0; i < N; ++i) { h_a[i] = dist(gen); h_b[i] = dist(gen); }

        // Serial dot product for correctness check and speedup baseline
        for (int i = 0; i < N; ++i) serial_dot += h_a[i] * h_b[i];
    }

    auto end_serial = std::chrono::high_resolution_clock::now();
    std::chrono::duration<double> serial_time = end_serial - start_serial;

    // ── Distribute data: scatter chunks to all ranks ──────────────────────────
    // Each rank gets chunk_size elements of A and B.
    // MPI_Scatter splits the array into equal parts and sends one to each rank.
    int chunk_size = N / size;
    std::vector<float> local_a(chunk_size), local_b(chunk_size);

    // MPI_Scatter(sendbuf, sendcount, sendtype,
    //             recvbuf, recvcount, recvtype, root, comm)
    MPI_Scatter(h_a.data(), chunk_size, MPI_FLOAT,
                local_a.data(), chunk_size, MPI_FLOAT, 0, MPI_COMM_WORLD);
    MPI_Scatter(h_b.data(), chunk_size, MPI_FLOAT,
                local_b.data(), chunk_size, MPI_FLOAT, 0, MPI_COMM_WORLD);

    // ── Each rank computes its partial dot product ─────────────────────────────
    // This is the parallel work — happening simultaneously on all cores.
    MPI_Barrier(MPI_COMM_WORLD);  // synchronize before timing parallel section
    auto start_par = std::chrono::high_resolution_clock::now();

    double local_dot = 0.0;
    for (int i = 0; i < chunk_size; ++i)
        local_dot += local_a[i] * local_b[i];

    // ── Reduce all partial results to rank 0 ──────────────────────────────────
    // MPI_Reduce(sendbuf, recvbuf, count, datatype, op, root, comm)
    // MPI_SUM tells MPI to add all local_dot values together.
    // Only rank 0 gets the result in global_dot.
    double global_dot = 0.0;
    MPI_Reduce(&local_dot, &global_dot, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);

    MPI_Barrier(MPI_COMM_WORLD);
    auto end_par = std::chrono::high_resolution_clock::now();
    std::chrono::duration<double> par_time = end_par - start_par;

    // ── Rank 0 reports results ────────────────────────────────────────────────
    if (rank == 0) {
        double error = std::abs(global_dot - serial_dot) / std::abs(serial_dot);
        double speedup = serial_time.count() / par_time.count();

        std::cout << "Vector size      : " << N << std::endl;
        std::cout << "Processes        : " << size << std::endl;
        std::cout << "Serial dot       : " << serial_dot << std::endl;
        std::cout << "Parallel dot     : " << global_dot << std::endl;
        std::cout << "Relative error   : " << error << std::endl;
        std::cout << "Serial time      : " << serial_time.count() << " s" << std::endl;
        std::cout << "Parallel time    : " << par_time.count()    << " s" << std::endl;
        std::cout << "Speedup          : " << speedup << "x" << std::endl;
    }

    MPI_Finalize();
    return 0;
}
```

### Compile and Run

```bash
mpic++ -O2 dot_product.cpp -o dot_product

# Run with different process counts
mpirun -np 1 ./dot_product
mpirun -np 2 ./dot_product
mpirun -np 4 ./dot_product
mpirun -np 8 ./dot_product   # only if nproc >= 8
```

> If you get a warning about oversubscription (more processes than cores), add `--oversubscribe`:
> ```bash
> mpirun --oversubscribe -np 8 ./dot_product
> ```
> This is fine for learning but won't give real speedup.

### Code Walkthrough

| Line / Call | Purpose |
|---|---|
| `MPI_Init` | Connects process to the MPI runtime — must be first MPI call |
| `MPI_Comm_rank` | Each process discovers its own rank |
| `MPI_Comm_size` | Each process learns total number of peers |
| `if (rank == 0)` generate data | Only one process generates the canonical vectors |
| Serial dot product (rank 0) | Baseline for correctness check and speedup denominator |
| `MPI_Scatter` | Rank 0 splits arrays and distributes one chunk to each rank — all ranks receive simultaneously |
| `local_dot` loop | **The parallel work** — runs on all cores simultaneously, each on its own chunk |
| `MPI_Reduce(..., MPI_SUM, ...)` | Collects and sums all `local_dot` values; result lands in `global_dot` on rank 0 |
| `MPI_Barrier` around timed section | Ensures all processes start and stop timing together |
| `MPI_Finalize` | Disconnects from runtime — must be last MPI call |

### Why This Is SPMD

Every rank runs the **same source file**. The `if (rank == 0)` branches are not different programs — they are different **data roles** within a single program. Rank 0 is data owner and reporter; other ranks are compute workers. The "different data" part is enforced by `MPI_Scatter`.

---

## 5. Pipeline Parallelism

### Concept

In a **pipeline**, each process performs a **different stage** of a computation and passes its output to the next process — like an assembly line. This is the MPMD-style logic within SPMD described earlier.

```
Rank 0          Rank 1          Rank 2          Rank 3
─────────       ─────────       ─────────       ─────────
Generate   →    Scale     →    Threshold  →    Aggregate
raw data        by factor       (filter)        & report
                               values > 0.5
```

Each stage runs on a different core simultaneously — while rank 1 is scaling the first batch, rank 0 is already generating the second batch.

### The Code

```cpp
#include <mpi.h>
#include <iostream>
#include <vector>
#include <random>
#include <algorithm>
#include <numeric>

// Pipeline with 4 fixed stages. Must be run with exactly -np 4.
// Stage 0: Generate random floats in [0, 1)
// Stage 1: Scale all values by 2.0 (range becomes [0, 2))
// Stage 2: Threshold — zero out values below 0.5
// Stage 3: Compute and report the mean of surviving values

int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (size != 4) {
        if (rank == 0)
            std::cerr << "This pipeline requires exactly 4 processes." << std::endl;
        MPI_Finalize();
        return 1;
    }

    const int N = 1000;       // number of elements per batch
    const int tag = 0;        // message tag (used to distinguish message types)

    // ── Stage 0: Generator ───────────────────────────────────────────────────
    if (rank == 0) {
        std::vector<float> data(N);
        std::mt19937 gen(99);
        std::uniform_real_distribution<float> dist(0.0f, 1.0f);

        for (int i = 0; i < N; ++i) data[i] = dist(gen);

        std::cout << "[Rank 0 | Generator ] First 5 values: ";
        for (int i = 0; i < 5; ++i) std::cout << data[i] << " ";
        std::cout << std::endl;

        // Send generated data downstream to rank 1
        // MPI_Send(buf, count, datatype, dest, tag, comm)
        MPI_Send(data.data(), N, MPI_FLOAT, 1, tag, MPI_COMM_WORLD);
    }

    // ── Stage 1: Scaler ──────────────────────────────────────────────────────
    else if (rank == 1) {
        std::vector<float> data(N);

        // Receive from rank 0 (upstream stage)
        // MPI_Recv(buf, count, datatype, source, tag, comm, status)
        MPI_Recv(data.data(), N, MPI_FLOAT, 0, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

        for (int i = 0; i < N; ++i) data[i] *= 2.0f;   // scale

        std::cout << "[Rank 1 | Scaler    ] First 5 values after scaling: ";
        for (int i = 0; i < 5; ++i) std::cout << data[i] << " ";
        std::cout << std::endl;

        // Send scaled data to rank 2
        MPI_Send(data.data(), N, MPI_FLOAT, 2, tag, MPI_COMM_WORLD);
    }

    // ── Stage 2: Threshold filter ────────────────────────────────────────────
    else if (rank == 2) {
        std::vector<float> data(N);
        MPI_Recv(data.data(), N, MPI_FLOAT, 1, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

        int kept = 0;
        for (int i = 0; i < N; ++i) {
            if (data[i] < 0.5f) data[i] = 0.0f;   // suppress values below threshold
            else kept++;
        }

        std::cout << "[Rank 2 | Threshold ] Values above 0.5: " << kept
                  << " / " << N << std::endl;

        // Send filtered data to rank 3
        MPI_Send(data.data(), N, MPI_FLOAT, 3, tag, MPI_COMM_WORLD);
    }

    // ── Stage 3: Aggregator ──────────────────────────────────────────────────
    else if (rank == 3) {
        std::vector<float> data(N);
        MPI_Recv(data.data(), N, MPI_FLOAT, 2, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

        // Mean of non-zero values
        double sum = 0.0;
        int count = 0;
        for (int i = 0; i < N; ++i) {
            if (data[i] > 0.0f) { sum += data[i]; count++; }
        }
        double mean = (count > 0) ? sum / count : 0.0;

        std::cout << "[Rank 3 | Aggregator] Mean of surviving values: "
                  << mean << " (count=" << count << ")" << std::endl;
    }

    MPI_Finalize();
    return 0;
}
```

### Compile and Run

```bash
mpic++ -O2 pipeline.cpp -o pipeline
mpirun -np 4 ./pipeline
```

### Expected Output

```
[Rank 0 | Generator ] First 5 values: 0.134 0.847 0.763 0.213 0.921
[Rank 1 | Scaler    ] First 5 values after scaling: 0.268 1.694 1.526 0.426 1.842
[Rank 2 | Threshold ] Values above 0.5: 748 / 1000
[Rank 3 | Aggregator] Mean of surviving values: 1.247 (count=748)
```

### Code Walkthrough

| Rank | Role | MPI Calls Used |
|---|---|---|
| 0 | Generates raw data | `MPI_Send` (to rank 1) |
| 1 | Scales values | `MPI_Recv` (from 0), `MPI_Send` (to 2) |
| 2 | Filters by threshold | `MPI_Recv` (from 1), `MPI_Send` (to 3) |
| 3 | Computes final aggregate | `MPI_Recv` (from 2), prints result |

**Why blocking Send/Recv works here:** Each stage waits for the previous stage to finish before proceeding. This creates natural synchronization along the pipeline without explicit barriers. The pipeline topology is a chain — no deadlock is possible because no two ranks are waiting on each other simultaneously (no circular dependency).

---

## 6. Master-Worker Pattern

### Concept

In the **master-worker** model, one process (rank 0 — the master) distributes **tasks** to worker processes and collects their **results**. Workers ask for work, do it, and report back. This is ideal when tasks have varying or unpredictable durations — it achieves natural **load balancing** because idle workers immediately get new work.

```
           ┌─────────────┐
           │   Master    │ ← rank 0: task queue, result collection
           │   (Rank 0)  │
           └──┬────┬──┬──┘
     task     │    │  │     result
    ┌──────◄──┘    │  └──►──────┐
    ▼              │             ▼
┌────────┐    ┌────────┐    ┌────────┐
│Worker 1│    │Worker 2│    │Worker 3│
│ Rank 1 │    │ Rank 2 │    │ Rank 3 │
└────────┘    └────────┘    └────────┘
```

### The Code

```cpp
#include <mpi.h>
#include <iostream>
#include <vector>
#include <cmath>
#include <iomanip>

// Tag constants — distinguish message purpose on the wire
const int TAG_TASK   = 1;   // master sending a task to worker
const int TAG_RESULT = 2;   // worker sending result back to master
const int TAG_STOP   = 3;   // master telling worker to shut down

// Simulated workload: compute sum of square roots up to 'n'
// The higher n is, the longer it takes — simulates variable task duration
double compute_task(int n) {
    double sum = 0.0;
    for (int i = 1; i <= n; ++i)
        sum += std::sqrt((double)i);
    return sum;
}

int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (size < 2) {
        std::cerr << "Need at least 2 processes (1 master + 1 worker)." << std::endl;
        MPI_Finalize(); return 1;
    }

    // ── MASTER (Rank 0) ───────────────────────────────────────────────────────
    if (rank == 0) {
        // Define task queue — each entry is a workload size
        // Larger numbers take longer, simulating heterogeneous jobs
        std::vector<int> tasks = {100000, 500000, 200000, 800000,
                                   300000, 100000, 700000, 400000,
                                   600000, 250000};
        int num_tasks = tasks.size();
        int num_workers = size - 1;
        int task_index = 0;

        std::cout << "Master: " << num_tasks << " tasks, "
                  << num_workers << " workers." << std::endl;

        // ── Seed each worker with its first task ──────────────────────────────
        // Send one task to each worker to get them started.
        // Workers 1..num_workers each get the first batch.
        for (int w = 1; w <= std::min(num_workers, num_tasks); ++w) {
            std::cout << "Master: Sending task " << task_index
                      << " (size=" << tasks[task_index] << ") to worker " << w << std::endl;
            MPI_Send(&tasks[task_index], 1, MPI_INT, w, TAG_TASK, MPI_COMM_WORLD);
            task_index++;
        }

        // ── Dynamic task distribution loop ────────────────────────────────────
        // Receive results and dispatch new tasks until all tasks are done.
        double total_result = 0.0;
        int tasks_done = 0;

        while (tasks_done < num_tasks) {
            double result;
            MPI_Status status;

            // Wait for ANY worker to send a result back (MPI_ANY_SOURCE)
            // This naturally handles load imbalance — whoever finishes first gets the next job
            MPI_Recv(&result, 1, MPI_DOUBLE, MPI_ANY_SOURCE, TAG_RESULT,
                     MPI_COMM_WORLD, &status);

            int worker = status.MPI_SOURCE;  // which worker just finished?
            total_result += result;
            tasks_done++;

            std::cout << "Master: Received result from worker " << worker
                      << " (tasks done: " << tasks_done << "/" << num_tasks << ")" << std::endl;

            // If more tasks remain, send next task to the worker that just finished
            if (task_index < num_tasks) {
                std::cout << "Master: Sending task " << task_index
                          << " (size=" << tasks[task_index] << ") to worker " << worker << std::endl;
                MPI_Send(&tasks[task_index], 1, MPI_INT, worker, TAG_TASK, MPI_COMM_WORLD);
                task_index++;
            }
        }

        // ── Send stop signal to all workers ───────────────────────────────────
        for (int w = 1; w <= num_workers; ++w) {
            int dummy = 0;
            MPI_Send(&dummy, 1, MPI_INT, w, TAG_STOP, MPI_COMM_WORLD);
        }

        std::cout << "\nMaster: All tasks complete. Total result = "
                  << std::fixed << std::setprecision(2) << total_result << std::endl;
    }

    // ── WORKERS (Rank 1, 2, 3, ...) ──────────────────────────────────────────
    else {
        while (true) {
            int task_size;
            MPI_Status status;

            // Block and wait for any message from master
            MPI_Recv(&task_size, 1, MPI_INT, 0, MPI_ANY_TAG,
                     MPI_COMM_WORLD, &status);

            // Check the tag to decide what to do
            if (status.MPI_TAG == TAG_STOP) {
                // Master says we're done
                std::cout << "Worker " << rank << ": Received stop signal. Exiting." << std::endl;
                break;
            }

            // TAG_TASK: do the work and send result back
            std::cout << "Worker " << rank << ": Computing task of size "
                      << task_size << "..." << std::endl;

            double result = compute_task(task_size);

            // Send result back to master (rank 0)
            MPI_Send(&result, 1, MPI_DOUBLE, 0, TAG_RESULT, MPI_COMM_WORLD);
        }
    }

    MPI_Finalize();
    return 0;
}
```

### Compile and Run

```bash
mpic++ -O2 master_worker.cpp -o master_worker

mpirun -np 3 ./master_worker   # 1 master + 2 workers
mpirun -np 5 ./master_worker   # 1 master + 4 workers
```

### Code Walkthrough

**Master logic:**
1. Builds a task queue with variable-size jobs
2. Seeds workers with their first task (so no worker starts idle)
3. Enters a loop: wait for any worker result → dispatch next task to that worker → repeat
4. When all tasks are done, sends `TAG_STOP` to every worker

**Worker logic:**
1. Blocks on `MPI_Recv` with `MPI_ANY_TAG` — it doesn't know if the next message is a task or stop signal
2. Checks `status.MPI_TAG` after receiving — branches on tag value
3. If task: computes, sends result back
4. If stop: breaks out of the loop and exits cleanly

**Key design insight — `MPI_ANY_SOURCE` on master:** The master uses `MPI_ANY_SOURCE` to receive from whichever worker finishes first. This is the load balancing mechanism — slow workers don't block fast ones from getting new tasks. Without this, you'd have to pre-assign tasks statically, causing fast workers to sit idle while slow ones finish.

---

## 7. Exercise 1: Amdahl's Law Table

### Background

**Amdahl's Law** quantifies the maximum speedup achievable when parallelizing a program that has a non-parallelizable serial fraction `f`:

$$S(n) = \frac{1}{f + \frac{1-f}{n}}$$

Where:
- `S(n)` = speedup with `n` processors
- `f` = fraction of execution time that is **serial** (cannot be parallelized)
- `1 - f` = fraction that is **parallel**

As `n → ∞`, `S → 1/f` — the serial fraction becomes the hard ceiling.

### Your Task

**Part A — Fill the theoretical table:**

Using the formula above, compute the theoretical speedup for each combination:

| Serial Fraction `f` | 1 core | 2 cores | 4 cores | 8 cores | 16 cores | Limit (∞ cores) |
|---|---|---|---|---|---|---|
| 0% (fully parallel) | 1.0 | | | | | |
| 10% | 1.0 | | | | | |
| 25% | 1.0 | | | | | |
| 50% | 1.0 | | | | | |
| 75% | 1.0 | | | | | |
| 90% | 1.0 | | | | | |

**Part B — Measure real speedup from the dot product:**

Re-run the dot product program from Section 4 with different core counts and fill this table:

| Processes | Wall time (s) | Speedup vs 1 process | Efficiency (Speedup / P) |
|---|---|---|---|
| 1 | | 1.0x | 1.0 |
| 2 | | | |
| 4 | | | |
| 8 | | | |

**Part C — Discussion questions:**

1. From your measurements in Part B, estimate the effective serial fraction `f` using Amdahl's formula. Does it match what you expect from the code?
2. At which point does adding more processes give diminishing returns? Why?
3. What does efficiency dropping below 1.0 mean physically?
4. What parts of the dot product code are serial? (Hint: look at rank 0's work.)

---

## 8. Exercise 2: Process Scheduler Simulation

### Background and Goal

You will simulate a simplified **OS process scheduler** using the master-worker MPI pattern. The master acts as the **OS scheduler** and workers are **CPU cores** executing processes from a ready queue.

The scheduler implements **Shortest Job First (SJF)** — it always dispatches the shortest pending job to the next available core. This is a classic scheduling algorithm that minimises average waiting time.

### What You Need to Build

- **Master (rank 0):** Maintains a queue of `Job` structs (each with an ID, a burst time, and an arrival time). Sorts the queue by burst time (SJF). Assigns jobs to workers as cores become free. Prints the scheduling order and per-job results.
- **Workers (rank 1..N):** Receive a job, simulate execution by running a busy loop proportional to burst time, return completion time to master.

### Starter Code

```cpp
#include <mpi.h>
#include <iostream>
#include <vector>
#include <algorithm>
#include <random>
#include <chrono>
#include <iomanip>

// ── Data structures ──────────────────────────────────────────────────────────
struct Job {
    int id;
    int burst_units;    // how many loop iterations to run (simulates CPU time)
    double arrival_time; // relative arrival time (seconds from t=0)
};

// Pack Job into an int array for MPI transmission
// MPI cannot send structs directly without defining a custom MPI datatype.
// Simplest approach: serialize to a plain array.
void pack_job(const Job& j, int* buf) {
    buf[0] = j.id;
    buf[1] = j.burst_units;
    // arrival_time not needed by workers
}

Job unpack_job(const int* buf) {
    return {buf[0], buf[1], 0.0};
}

// Simulate a process running: busy loop to waste CPU time
void simulate_execution(int burst_units) {
    volatile double sink = 0.0;
    for (int i = 0; i < burst_units * 10000; ++i)
        sink += i * 0.000001;
    (void)sink;
}

const int TAG_JOB    = 1;
const int TAG_DONE   = 2;
const int TAG_STOP   = 3;
const int BUF_SIZE   = 4;

int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (size < 2) {
        if (rank == 0) std::cerr << "Need at least 2 processes." << std::endl;
        MPI_Finalize(); return 1;
    }

    // ── MASTER ────────────────────────────────────────────────────────────────
    if (rank == 0) {
        // Generate a random job queue
        std::mt19937 gen(2024);
        std::uniform_int_distribution<int> burst_dist(1, 20);
        std::uniform_real_distribution<double> arrive_dist(0.0, 5.0);

        int num_jobs = 12;
        std::vector<Job> job_queue(num_jobs);
        for (int i = 0; i < num_jobs; ++i)
            job_queue[i] = {i, burst_dist(gen), arrive_dist(gen)};

        // TODO: sort job_queue by burst_units ascending (SJF policy)

        // Print initial queue
        std::cout << "\n═══ Process Scheduler (SJF) ══════════════════════════\n";
        std::cout << std::left << std::setw(8) << "Job ID"
                               << std::setw(15) << "Burst Units"
                               << std::setw(12) << "Arrival" << "\n";
        std::cout << std::string(40, '─') << "\n";
        for (auto& j : job_queue)
            std::cout << std::setw(8) << j.id
                      << std::setw(15) << j.burst_units
                      << std::setw(12) << std::fixed << std::setprecision(2)
                      << j.arrival_time << "\n";
        std::cout << std::string(40, '═') << "\n\n";

        int num_workers = size - 1;
        int job_index = 0;
        int jobs_done = 0;

        // TODO: seed each worker with its first job (same pattern as master-worker example)

        // TODO: main scheduling loop
        //   - Receive completion from any worker (MPI_ANY_SOURCE, TAG_DONE)
        //     Worker sends back: [job_id, worker_rank]
        //   - Print "Job X completed by Worker Y"
        //   - If more jobs remain, dispatch next job to that worker
        //   - When all jobs done, send TAG_STOP to all workers

        // TODO: print summary table
        //   Columns: Job ID | Assigned Worker | Burst Units | Status

        std::cout << "\nScheduler: All jobs completed.\n";
    }

    // ── WORKERS ───────────────────────────────────────────────────────────────
    else {
        while (true) {
            int buf[BUF_SIZE];
            MPI_Status status;
            MPI_Recv(buf, BUF_SIZE, MPI_INT, 0, MPI_ANY_TAG, MPI_COMM_WORLD, &status);

            if (status.MPI_TAG == TAG_STOP) break;

            Job j = unpack_job(buf);
            std::cout << "  [Worker " << rank << "] Executing Job "
                      << j.id << " (burst=" << j.burst_units << ")" << std::endl;

            simulate_execution(j.burst_units);

            // TODO: send completion message back to master
            // buf should contain: [j.id, rank]
        }
    }

    MPI_Finalize();
    return 0;
}
```

### Compile and Run

```bash
mpic++ -O2 scheduler.cpp -o scheduler
mpirun -np 4 ./scheduler   # 1 scheduler + 3 CPU cores
mpirun -np 6 ./scheduler   # 1 scheduler + 5 CPU cores
```

### Hints

> **Hint 1 — Sorting for SJF:**
> Use `std::sort` with a lambda comparator on `burst_units`. The job with the smallest burst time should be dispatched first.

> **Hint 2 — Seeding workers initially:**
> Mirror the master-worker example: loop `w` from 1 to `min(num_workers, num_jobs)`, call `pack_job` into a buffer, then `MPI_Send` with `TAG_JOB`.

> **Hint 3 — Completion message format:**
> Workers should send a 2-element int array: `{job_id, worker_rank}`. Master receives it, reads the job ID, increments `jobs_done`, and uses `status.MPI_SOURCE` to know which worker to dispatch next.

> **Hint 4 — Preventing over-dispatch:**
> Track `in_flight` (how many workers currently have a job). Only the first `min(num_workers, num_jobs)` workers get seeded — don't send to workers that don't exist.

> **Hint 5 — Summary table:**
> Keep a `std::map<int, int>` mapping job ID → worker rank as jobs complete. After all jobs are done, iterate the original `job_queue` and print the table in order.

### Expected Output Shape

```
═══ Process Scheduler (SJF) ══════════════════════════
Job ID  Burst Units    Arrival
────────────────────────────────────────
3       1              1.23
7       2              4.01
...
════════════════════════════════════════

  [Worker 1] Executing Job 3 (burst=1)
  [Worker 2] Executing Job 7 (burst=2)
  [Worker 3] Executing Job 0 (burst=3)
Master: Job 3 completed by Worker 1
Master: Dispatching Job 11 to Worker 1
...
Master: All jobs completed.

 Final Schedule Summary
────────────────────────────────────────
 Job 3  │ Worker 1 │ Burst  1 │ Done ✅
 Job 7  │ Worker 2 │ Burst  2 │ Done ✅
...
```

### Discussion Questions

1. How does the number of workers affect total throughput? At what point does adding workers give no benefit for 12 jobs?
2. Compare SJF to FIFO (arrival-order dispatch): how would you change the code? What happens to average wait time?
3. What happens if a worker crashes mid-task in a real MPI program? How would you detect and recover from this?
4. The `simulate_execution` function uses `volatile` — why? What would happen without it?

---

*Lab prepared for Systems Programming / Parallel Computing course.*
