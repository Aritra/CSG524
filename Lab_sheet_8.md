# Lab: Introduction to NVIDIA GPU Programming with CUDA

---

## Prerequisites

- Ubuntu 24.04
- NVIDIA GT730 2GB GPU (Kepler architecture, Compute Capability 3.5)
- `build-essential`, `gcc`, `g++` installed

---

## Part 1: Installing NVCC (CUDA Compiler) on Ubuntu 24.04

### Step 1: Verify your GPU is detected

```bash
lspci | grep -i nvidia
```

Expected output should show something like:
```
01:00.0 VGA compatible controller: NVIDIA Corporation GK208B [GeForce GT 730] (rev a1)
```

### Step 2: Install NVIDIA drivers

```bash
sudo apt update
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

After reboot, verify the driver is loaded:

```bash
nvidia-smi
```

### Step 3: Install CUDA Toolkit

The GT730 supports up to **CUDA 12.x** (Compute Capability 3.5). Install the toolkit:

```bash
sudo apt install -y nvidia-cuda-toolkit
```

Verify the installation:

```bash
nvcc --version
```

Expected output:
```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on ...
Cuda compilation tools, release 12.x, V12.x.xxx
```

> **Note:** If `nvidia-cuda-toolkit` from apt gives an older version, you can install directly from [developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads) by selecting Linux → x86_64 → Ubuntu → 24.04 → deb(network).

---

### ⚠️ Alternative: LeetGPU Playground (if local install fails)

If driver installation fails (common in VMs or dual-GPU setups), use the browser-based **LeetGPU Playground**:

1. Go to [https://leetgpu.com](https://leetgpu.com)
2. Select **Playground** from the menu
3. Choose a GPU instance (free tier available)
4. Write and run CUDA `.cu` code directly in the browser — no local install needed

This is a fully functional CUDA environment and is recommended as a fallback for the entire lab.

---

## Part 2: Verify Compilation with a GPU Kernel Sample

Create a file called `hello_gpu.cu`:

```cuda
#include <stdio.h>

__global__ void helloKernel() {
    printf("Hello from GPU thread %d in block %d\n", threadIdx.x, blockIdx.x);
}

int main() {
    // Launch 2 blocks, 4 threads each
    helloKernel<<<2, 4>>>();
    cudaDeviceSynchronize();
    return 0;
}
```

Compile and run:

```bash
nvcc -o hello_gpu hello_gpu.cu
./hello_gpu
```

Expected output (order may vary — GPU threads run in parallel):
```
Hello from GPU thread 0 in block 0
Hello from GPU thread 1 in block 0
Hello from GPU thread 2 in block 0
Hello from GPU thread 3 in block 0
Hello from GPU thread 0 in block 1
...
```

If you see this output, your CUDA installation is working correctly. ✅

---

## Part 3: Vector Addition on the GPU

### The Concept: Thread and Block Indexing

Before writing the kernel, it is essential to understand how CUDA organizes parallel work.

CUDA launches threads in a two-level hierarchy:

```
Grid
├── Block 0 : [Thread 0, Thread 1, ..., Thread N-1]
├── Block 1 : [Thread 0, Thread 1, ..., Thread N-1]
└── ...
```

Each thread computes a **unique global index** using:

```cuda
int i = blockIdx.x * blockDim.x + threadIdx.x;
```

| Variable | Meaning |
|---|---|
| `threadIdx.x` | Thread's index within its block (0 to blockDim.x - 1) |
| `blockIdx.x` | Block's index within the grid (0 to gridDim.x - 1) |
| `blockDim.x` | Number of threads per block |

**Example:** With `blockDim.x = 4` and thread 2 in block 1:
```
i = 1 * 4 + 2 = 6
```
So this thread handles element index 6 of the array.

**Choosing threads per block:** A common choice is **256** threads per block. It must be a multiple of 32 (the warp size). The number of blocks is then computed as:

```cuda
int threadsPerBlock = 256;
int numBlocks = (n + threadsPerBlock - 1) / threadsPerBlock;
```

The `+ threadsPerBlock - 1` ensures we always round **up** — so even if `n` is not a perfect multiple of 256, we still cover all elements. This is why the kernel must guard with `if (i < n)` to avoid out-of-bounds access.

---

### Vector Addition: CPU + GPU with Verification and Speedup

Create `vector_add.cu`:

```cuda
#include <iostream>
#include <vector>
#include <random>
#include <chrono>
#include <cmath>
#include <cuda_runtime.h>

// -------------------------------------------------------
// GPU KERNEL: each thread adds one pair of elements
// -------------------------------------------------------
__global__ void vectorAddKernel(const float* a, const float* b, float* result, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        result[i] = a[i] + b[i];
    }
}

// -------------------------------------------------------
// CPU VERSION: sequential loop
// -------------------------------------------------------
void vectorAddCPU(const std::vector<float>& a,
                  const std::vector<float>& b,
                  std::vector<float>& result) {
    for (size_t i = 0; i < a.size(); ++i) {
        result[i] = a[i] + b[i];
    }
}

int main() {
    int n = 1 << 20; // ~1 million elements

    // -------------------------------------------------------
    // 1. Allocate and fill host (CPU) vectors with random data
    // -------------------------------------------------------
    std::vector<float> h_a(n), h_b(n), h_result_cpu(n), h_result_gpu(n);

    std::mt19937 gen(42);
    std::uniform_real_distribution<float> dist(1.0f, 10.0f);
    for (int i = 0; i < n; ++i) {
        h_a[i] = dist(gen);
        h_b[i] = dist(gen);
    }

    // -------------------------------------------------------
    // 2. Allocate memory on the GPU (device)
    // -------------------------------------------------------
    float *d_a, *d_b, *d_result;
    cudaMalloc((void**)&d_a,      n * sizeof(float));
    cudaMalloc((void**)&d_b,      n * sizeof(float));
    cudaMalloc((void**)&d_result, n * sizeof(float));

    // -------------------------------------------------------
    // 3. Copy input data from CPU to GPU
    // -------------------------------------------------------
    cudaMemcpy(d_a, h_a.data(), n * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b.data(), n * sizeof(float), cudaMemcpyHostToDevice);

    // -------------------------------------------------------
    // 4. Configure and launch the GPU kernel
    // -------------------------------------------------------
    int threadsPerBlock = 256;
    int numBlocks = (n + threadsPerBlock - 1) / threadsPerBlock;

    auto start_gpu = std::chrono::high_resolution_clock::now();

    vectorAddKernel<<<numBlocks, threadsPerBlock>>>(d_a, d_b, d_result, n);
    cudaDeviceSynchronize(); // wait for GPU to finish

    auto end_gpu = std::chrono::high_resolution_clock::now();

    // -------------------------------------------------------
    // 5. Copy result from GPU back to CPU
    // -------------------------------------------------------
    cudaMemcpy(h_result_gpu.data(), d_result, n * sizeof(float), cudaMemcpyDeviceToHost);

    // -------------------------------------------------------
    // 6. Run CPU version and time it
    // -------------------------------------------------------
    auto start_cpu = std::chrono::high_resolution_clock::now();
    vectorAddCPU(h_a, h_b, h_result_cpu);
    auto end_cpu = std::chrono::high_resolution_clock::now();

    std::chrono::duration<double> gpu_time = end_gpu - start_gpu;
    std::chrono::duration<double> cpu_time = end_cpu - start_cpu;

    // -------------------------------------------------------
    // 7. Verify results match
    // -------------------------------------------------------
    bool correct = true;
    for (int i = 0; i < n; ++i) {
        if (std::abs(h_result_cpu[i] - h_result_gpu[i]) > 1e-5f) {
            correct = false;
            break;
        }
    }

    // -------------------------------------------------------
    // 8. Print results
    // -------------------------------------------------------
    std::cout << "Vector size      : " << n << std::endl;
    std::cout << "GPU time         : " << gpu_time.count() << " s" << std::endl;
    std::cout << "CPU time         : " << cpu_time.count() << " s" << std::endl;
    std::cout << "Speedup (CPU/GPU): " << (cpu_time.count() / gpu_time.count()) << "x" << std::endl;
    std::cout << "Verification     : " << (correct ? "PASSED" : "FAILED") << std::endl;

    // -------------------------------------------------------
    // 9. Free GPU memory
    // -------------------------------------------------------
    cudaFree(d_a);
    cudaFree(d_b);
    cudaFree(d_result);

    return 0;
}
```

Compile and run:

```bash
nvcc -O2 -o vector_add vector_add.cu
./vector_add
```

---

## Part 4: Code Walkthrough — Step by Step

### 4.1 The GPU Kernel

```cuda
__global__ void vectorAddKernel(const float* a, const float* b, float* result, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        result[i] = a[i] + b[i];
    }
}
```

- `__global__` marks this function as a GPU kernel callable from CPU code.
- Every thread computes its own unique index `i` and processes exactly one pair of elements.
- `if (i < n)` guards against out-of-bounds access when `n` is not a perfect multiple of `threadsPerBlock`.

### 4.2 Host Memory Allocation

```cuda
std::vector<float> h_a(n), h_b(n), h_result_cpu(n), h_result_gpu(n);
```

These vectors live in **CPU RAM** (the "host"). The `h_` prefix is a naming convention meaning "host". The GPU cannot directly read these — they must be explicitly transferred.

### 4.3 Device Memory Allocation

```cuda
float *d_a, *d_b, *d_result;
cudaMalloc((void**)&d_a, n * sizeof(float));
```

`cudaMalloc` allocates memory on the **GPU (device)**. The `d_` prefix means "device". This memory is separate from CPU RAM and is much faster for GPU threads to access.

### 4.4 Copying Data to the GPU

```cuda
cudaMemcpy(d_a, h_a.data(), n * sizeof(float), cudaMemcpyHostToDevice);
```

Copies `n` floats from CPU memory to GPU memory. The direction flag `cudaMemcpyHostToDevice` tells the runtime which way the transfer flows. This transfer goes over the PCIe bus and is often the bottleneck for small workloads.

### 4.5 Kernel Launch

```cuda
vectorAddKernel<<<numBlocks, threadsPerBlock>>>(d_a, d_b, d_result, n);
```

The `<<<numBlocks, threadsPerBlock>>>` syntax is unique to CUDA. It launches `numBlocks × threadsPerBlock` threads simultaneously on the GPU. For `n = 1M` and `threadsPerBlock = 256`, this starts **4096 blocks × 256 threads = 1,048,576 threads** in one line.

### 4.6 Synchronization

```cuda
cudaDeviceSynchronize();
```

Kernel launches are **asynchronous** — the CPU continues while the GPU works. `cudaDeviceSynchronize()` blocks the CPU until all GPU work finishes. Without this, timing would be wrong and results would be incomplete when copied back.

### 4.7 Copying Results Back

```cuda
cudaMemcpy(h_result_gpu.data(), d_result, n * sizeof(float), cudaMemcpyDeviceToHost);
```

Direction reversed: `cudaMemcpyDeviceToHost` copies from GPU → CPU so we can inspect the results.

### 4.8 Freeing GPU Memory

```cuda
cudaFree(d_a);
cudaFree(d_b);
cudaFree(d_result);
```

GPU memory is not managed by the CPU allocator. Always free it explicitly with `cudaFree`, similar to how you would call `free()` for a `malloc`.

---

## Part 5: Vector Multiplication — GPU vs CPU

The kernel changes by exactly one character. Everything else is identical to vector addition. Create `vector_mul.cu`:

```cuda
#include <iostream>
#include <vector>
#include <random>
#include <chrono>
#include <cmath>
#include <cuda_runtime.h>

// GPU kernel: element-wise multiplication
__global__ void vectorMultiplyKernel(const float* a, const float* b, float* result, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        result[i] = a[i] * b[i];  // Only this line changes from addition
    }
}

// CPU version
void vectorMultiplyCPU(const std::vector<float>& a, const std::vector<float>& b,
                        std::vector<float>& result) {
    for (size_t i = 0; i < a.size(); ++i) {
        result[i] = a[i] * b[i];
    }
}

int main() {
    int n = 1 << 20;

    std::vector<float> h_a(n), h_b(n), h_result_cpu(n), h_result_gpu(n);

    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_real_distribution<float> dist(1.0f, 10.0f);
    for (int i = 0; i < n; ++i) { h_a[i] = dist(gen); h_b[i] = dist(gen); }

    float *d_a, *d_b, *d_result;
    cudaMalloc((void**)&d_a,      n * sizeof(float));
    cudaMalloc((void**)&d_b,      n * sizeof(float));
    cudaMalloc((void**)&d_result, n * sizeof(float));

    cudaMemcpy(d_a, h_a.data(), n * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b.data(), n * sizeof(float), cudaMemcpyHostToDevice);

    int threadsPerBlock = 256;
    int numBlocks = (n + threadsPerBlock - 1) / threadsPerBlock;

    auto start_gpu = std::chrono::high_resolution_clock::now();
    vectorMultiplyKernel<<<numBlocks, threadsPerBlock>>>(d_a, d_b, d_result, n);
    cudaDeviceSynchronize();
    auto end_gpu = std::chrono::high_resolution_clock::now();

    cudaMemcpy(h_result_gpu.data(), d_result, n * sizeof(float), cudaMemcpyDeviceToHost);

    auto start_cpu = std::chrono::high_resolution_clock::now();
    vectorMultiplyCPU(h_a, h_b, h_result_cpu);
    auto end_cpu = std::chrono::high_resolution_clock::now();

    std::chrono::duration<double> gpu_time = end_gpu - start_gpu;
    std::chrono::duration<double> cpu_time = end_cpu - start_cpu;

    bool correct = true;
    for (int i = 0; i < n; ++i)
        if (std::abs(h_result_cpu[i] - h_result_gpu[i]) > 1e-5f) { correct = false; break; }

    std::cout << "Vector size      : " << n << std::endl;
    std::cout << "GPU time         : " << gpu_time.count() << " s" << std::endl;
    std::cout << "CPU time         : " << cpu_time.count() << " s" << std::endl;
    std::cout << "Speedup (CPU/GPU): " << (cpu_time.count() / gpu_time.count()) << "x" << std::endl;
    std::cout << "Verification     : " << (correct ? "PASSED" : "FAILED") << std::endl;

    cudaFree(d_a); cudaFree(d_b); cudaFree(d_result);
    return 0;
}
```

Compile and run:

```bash
nvcc -O2 -o vector_mul vector_mul.cu
./vector_mul
```

**Observation:** The parallel structure — `cudaMalloc`, `cudaMemcpy`, kernel launch, sync, copy back, `cudaFree` — is identical. Only the arithmetic inside the kernel changes. This is the reusable skeleton of nearly every CUDA program.

---

## Student Tasks

### Task 1: Matrix Multiplication (GPU vs CPU, Speedup Comparison)

**Goal:** Write a CUDA program that multiplies two square matrices of size `N×N`, compares results with a CPU reference, and records speedup across different sizes.

**Hints for the kernel:**

For matrix multiplication, each output element `C[row][col]` is a dot product of a row from `A` and a column from `B`. Use **2D thread indexing**:

```cuda
__global__ void matMulKernel(const float* A, const float* B, float* C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;

    if (row < N && col < N) {
        float sum = 0.0f;
        for (int k = 0; k < N; ++k) {
            sum += A[row * N + k] * B[k * N + col];
        }
        C[row * N + col] = sum;
    }
}
```

Launch with a 2D grid:

```cuda
dim3 threadsPerBlock(16, 16);   // 16x16 = 256 threads per block
dim3 numBlocks((N + 15) / 16, (N + 15) / 16);
matMulKernel<<<numBlocks, threadsPerBlock>>>(d_A, d_B, d_C, N);
```

**Test with these sizes and fill in the table:**

| Matrix Size | CPU Time (s) | GPU Time (s) | Speedup |
|---|---|---|---|
| 128 × 128 | | | |
| 256 × 256 | | | |
| 512 × 512 | | | |
| 1024 × 1024 | | | |

**Questions to answer in your report:**
1. At what matrix size does the GPU start to outperform the CPU? Why does the crossover happen there?
2. Why is the GPU slower for very small matrices even though it has more compute units?
3. Matrix multiplication has O(N³) compute but only O(N²) memory accesses. How does this ratio (called **arithmetic intensity**) relate to GPU efficiency?

---

### Task 2 (Extension): 2D Convolution

**Goal:** Implement 2D convolution on the GPU. Given an input matrix `I` of size `H×W` and a convolution kernel `K` of size `kH×kW`, compute output `O` where:

```
O[i][j] = sum over ki, kj of: I[i + ki][j + kj] * K[ki][kj]
```

**Test cases to benchmark:**

| Input Size | Kernel Size | CPU Time (s) | GPU Time (s) | Speedup |
|---|---|---|---|---|
| 512 × 512 | 3 × 3 | | | |
| 512 × 512 | 5 × 5 | | | |
| 1024 × 1024 | 7 × 7 | | | |

**Extension challenge:** Load the convolution kernel into CUDA **shared memory** (fast on-chip scratchpad) and compare performance vs. reading it from global memory on every access. Use `__shared__ float sharedK[kH][kW];` inside the kernel.

---

## Summary: The CUDA Programming Pattern

Every CUDA program follows the same 6-step skeleton:

```
1. Allocate host memory         →  std::vector / malloc
2. Allocate device memory       →  cudaMalloc
3. Copy Host → Device           →  cudaMemcpy(..., cudaMemcpyHostToDevice)
4. Launch kernel                →  myKernel<<<blocks, threads>>>(...)
5. Copy Device → Host           →  cudaMemcpy(..., cudaMemcpyDeviceToHost)
6. Free device memory           →  cudaFree
```

Understanding **thread index computation** (`blockIdx.x * blockDim.x + threadIdx.x`) is the single most important concept in CUDA — it is the mapping from each parallel thread to a unique data element.
