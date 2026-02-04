

## **Lab Sheet 3: Dynamic Memory Allocation and Function Calls in RISC-V**

---

## **Objective**

After completing this lab, students should be able to:

* Allocate memory dynamically in RARS
* Use heap memory safely
* Understand how `malloc`-like behavior maps to syscalls
* Write and call functions in assembly
* Understand `jal`, `ret`, stack usage, and recursion
* Pass arguments efficiently
* Implement recursive and iterative algorithms

---

# **Part 1 — Dynamic Memory Allocation**

---

## **1.1 Why Dynamic Allocation?**

Static memory (`.data`) is fixed at compile time.

Dynamic allocation allows:

* Runtime-sized arrays
* Unknown input sizes
* Linked data structures
* Heap-based memory management

---

## **1.2 Syscall for Dynamic Allocation in RARS**

### Allocate Heap Memory

```
a7 = 9   (sbrk syscall)
a0 = number of bytes requested
ecall
```

Return value:

```
a0 = base address of allocated memory
```

This behaves similar to:

$$
\text{ptr} = \text{malloc(size)}
$$

---

## **1.3 How compilers translate malloc**

C code:

```c
float *arr = malloc(n * 4);
```

Assembly conceptually becomes:

```assembly
mul t0, n, 4
mv  a0, t0
li  a7, 9
ecall
mv  s0, a0    # pointer
```

---

## **1.4 Important Cautions**

### ⚠ Always remember:

### 1. Allocation size must be in bytes

* float → 4 bytes
* double → 8 bytes

### 2. Save returned pointer

`a0` gets overwritten easily.

### 3. Avoid invalid memory

* Do not exceed allocated size
* Always use correct offset increments

### 4. RARS does NOT free memory

There is no `free()`.

---

# **1.5 Example — Mean of n Floating Point Numbers**

### Program:

1. Read integer `n`
2. Allocate `n × 4` bytes
3. Read `n` floats
4. Compute mean
5. Print mean

---

## ✅ Working RARS Code

```assembly
.data
promptN: .asciz "Enter n: "
promptF: .asciz "Enter float: "
newline: .asciz "\n"

.text
.globl main
main:

# ----- read n -----
li a7, 4
la a0, promptN
ecall

li a7, 5          # read int
ecall
mv s1, a0         # s1 = n

# ----- allocate memory (n * 4 bytes) -----
li t0, 4
mul a0, s1, t0
li a7, 9          # sbrk
ecall
mv s0, a0         # base pointer

# ----- input loop -----
li t1, 0          # i
fmv.s.x f1, zero  # sum = 0

input_loop:
beq t1, s1, compute_mean

li a7, 4
la a0, promptF
ecall

li a7, 6          # read float
ecall             # result in fa0

fsw fa0, 0(s0)
fadd.s f1, f1, fa0

addi s0, s0, 4
addi t1, t1, 1
j input_loop

compute_mean:

fcvt.s.w f2, s1
fdiv.s f3, f1, f2

fmv.s fa0, f3
li a7, 2          # print float
ecall

li a7, 10
ecall
```

---

---

# **Part 2 — Function Calls in RISC-V**

---

## **2.1 Jump vs Jump-and-Link**

### Jump

```
j label
```

* Control transfers
* No return possible

---

### Jump-and-Link

```
jal label
```

* Saves return address in `ra`
* Enables returning

---

### Why linking?

Because:

$$
\text{return address} = \text{PC} + 4
$$

Stored in `ra` so we can later:

```
ret   (jalr x0, 0(ra))
```

---

## **2.2 One-level function (no stack)**

```assembly
jal add_func

add_func:
add a0, a0, a1
ret
```

Works only if:

* No nested calls
* No recursion

---

## **2.3 Recursive calls → Need Stack**

### Why?

Each call must store:

* return address
* local variables

---

### Stack usage

```
addi sp, sp, -8
sw   ra, 4(sp)
sw   a0, 0(sp)
...
lw   ra, 4(sp)
addi sp, sp, 8
ret
```

---

## **2.4 Argument Passing**

Registers:

| Type   | Registers |
| ------ | --------- |
| args   | a0–a7     |
| return | a0        |
| saved  | s0–s11    |

If arguments exceed 8:

* pass pointer to array

Example:

```assembly
la a0, arr
jal process_array
```

---

# **2.5 Example — Factorial (recursive)**

### Computes:

$$
n! = n \times (n-1)!
$$

---

## ✅ Correct RARS Code

```assembly
.data
nval: .word 5

.text
.globl main

main:
la t0, nval
lw a0, 0(t0)

jal fact

li a7, 1
ecall

li a7, 10
ecall

# ---------- factorial ----------
fact:

addi sp, sp, -8
sw ra, 4(sp)
sw a0, 0(sp)

li t0, 1
ble a0, t0, base_case

addi a0, a0, -1
jal fact

lw t1, 0(sp)
mul a0, a0, t1
j end_fact

base_case:
li a0, 1

end_fact:
lw ra, 4(sp)
addi sp, sp, 8
ret
```

---

---

# **Student Problems**

---

## **Problem 1 — Extended Mean**

Modify the mean program to:

### Tasks

* Use double precision
* Compute standard deviation

Formula:

$$
\mu = \frac{1}{n}\sum x_i
$$

$$
\sigma = \sqrt{\frac{1}{n}\sum (x_i - \mu)^2}
$$

---

## **Problem 2 — Binary Search (Integers Only)**

Given a sorted array:

```
1 3 5 7 9 11 15
```

Write code to:

* search a key
* return index or -1

Algorithm:

1. low = 0
2. high = n-1
3. mid = (low+high)/2
4. compare
5. repeat

---

---

# **Expected Learning Outcome**

After this lab, students should:

* Allocate heap memory correctly
* Understand stack and recursion
* Write modular assembly functions
* Use argument registers effectively
* Translate algorithms to assembly

---

**End of Lab Sheet 3**
