# **CS G524 – Advanced Computer Architecture**

## **Lab Sheet 1: Introduction to RISC-V Assembly Programming**

**Base Platform:** RISC-V (RV32)
**Simulator:** RARS

---

## **Objective**

The objective of this lab is to familiarize students with:

* Basic structure of RISC-V assembly programs
* Console input/output using system calls
* Integer arithmetic and expression evaluation
* Efficient vs inefficient register usage
* Comparison, memory access, and simple array processing

This lab forms the foundation for later experiments on **Instruction Level Parallelism (ILP)**.

---

## **1. Basic RISC-V Assembly Code Structure**

A RISC-V assembly program typically consists of **two main sections**:

```assembly
.data
    # Data declarations (variables, arrays, strings)

.text
    .globl main
main:
    # Instructions start here
```

### Key Points

* `.data` section stores static data
* `.text` section stores executable instructions
* `main:` is the program entry point
* Comments start with `#`

---

## **2. Basic Input and Output (Integers & Strings)**

### Printing a String

```assembly
.data
msg: .asciiz "Hello, RISC-V!\n"

.text
.globl main
main:
    li a7, 4        # syscall: print string
    la a0, msg
    ecall
```

---

### Printing an Integer

```assembly
    li a0, 10       # integer to print
    li a7, 1        # syscall: print integer
    ecall
```

---

### Taking Integer Input from Terminal

```assembly
    li a7, 5        # syscall: read integer
    ecall
    # input value is now in a0
```

---

### Exiting the Program

```assembly
    li a7, 10       # syscall: exit
    ecall
```

---

## **3. Arithmetic Operations and Expressions**

### Basic Arithmetic Operations

```assembly
    add t0, t1, t2   # t0 = t1 + t2
    sub t3, t1, t2   # t3 = t1 - t2
    mul t4, t1, t2   # t4 = t1 * t2
    div t5, t1, t2   # t5 = t1 / t2
    rem t6, t1, t2   # t6 = t1 % t2
```

---

### Example: Polynomial of Order 3

Compute:

[
P(x) = ax^3 + bx^2 + cx + d
]

#### **Suboptimal Code (Poor Register Reuse)**

```assembly
    mul t0, a0, a0      # x^2
    mul t1, t0, a0      # x^3
    mul t2, t1, s0      # a*x^3
    mul t3, t0, s1      # b*x^2
    mul t4, a0, s2      # c*x
    add t5, t2, t3
    add t6, t5, t4
    add t6, t6, s3      # + d
```

**Issues:**

* Excessive temporary registers
* Longer dependency chains
* Higher register pressure

---

#### **Optimal Code (Register Reuse & Fewer Instructions)**

```assembly
    mul t0, a0, a0      # x^2
    mul t0, t0, a0      # x^3
    mul t0, t0, s0      # a*x^3
    mul t1, a0, a0      # x^2
    mul t1, t1, s1      # b*x^2
    add t0, t0, t1
    mul t1, a0, s2      # c*x
    add t0, t0, t1
    add t0, t0, s3      # + d
```

**Why this is better:**

* Reduced number of live registers
* Better suited for ILP and pipelining
* Easier for the compiler/hardware to optimize

---

## **4. Comparisons, Memory Access, and Arrays**

### Comparison Instructions

```assembly
    beq t0, t1, label   # if t0 == t1
    blt t0, t1, label   # if t0 < t1
    bgt t0, t1, label   # if t0 > t1
```

---

### Loading and Storing from Memory

```assembly
    lw t0, 0(t1)    # load word from memory
    sw t0, 0(t1)    # store word to memory
```

---

### Example: Printing a Hardcoded Integer Array

**Array:** `1 2 3 4 5`

```assembly
.data
arr: .word 1, 2, 3, 4, 5
n:   .word 5

.text
.globl main
main:
    la t0, arr
    li t1, 5

print_loop:
    lw a0, 0(t0)
    li a7, 1
    ecall

    li a0, 32       # space
    li a7, 11
    ecall

    addi t0, t0, 4
    addi t1, t1, -1
    bnez t1, print_loop
```

---

### Cumulative Sum of Array

**Input:** `1 2 3 4 5`
**Output:** `1 3 6 10 15`

```assembly
    la t0, arr
    li t1, 5
    li t2, 0        # cumulative sum

sum_loop:
    lw t3, 0(t0)
    add t2, t2, t3

    mv a0, t2
    li a7, 1
    ecall

    li a0, 32
    li a7, 11
    ecall

    addi t0, t0, 4
    addi t1, t1, -1
    bnez t1, sum_loop
```

---

## **Student Tasks (To Be Submitted)**

### **Task 1: Power Calculation**

Write a RISC-V program that:

1. Takes two integers `x` and `y` as input from the user
2. Computes (x^y) using a loop (do **not** use any power instruction)
3. Prints the result

---

### **Task 2: Range of an Array**

Given a **hardcoded word array** of fixed size:

1. Traverse the array
2. Find the **maximum** and **minimum** elements
3. Compute and print the **range**:

[
\text{Range} = \text{max} - \text{min}
]

---

## **Expected Learning Outcome**

After completing this lab, students should be comfortable with:

* Writing complete RISC-V programs
* Using registers efficiently
* Performing arithmetic and memory operations
* Understanding how low-level code structure impacts performance

---

**End of Lab Sheet 1**
