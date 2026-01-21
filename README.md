Below is a clean, student-friendly **README.md** you can directly use in your GitHub repository.

---

# CS G524 – Advanced Computer Architecture

**BITS Pilani, Hyderabad Campus (M.E./M.Tech)**

This repository contains course materials, notes, examples, and programming exercises for **CS G524: Advanced Computer Architecture**, offered to **Master’s degree students at BITS Pilani, Hyderabad Campus**.

The focus of this course is on **quantitative evaluation of computer architectures**, with special emphasis on **Instruction Level Parallelism (ILP)** and **RISC-V**–based design and programming.

---

## 📚 Course Reference

The primary textbook followed in this course is:

> **Computer Architecture: A Quantitative Approach**
> *John L. Hennessy & David A. Patterson*

This book will be used extensively for:

* Performance analysis
* Pipelining and hazards
* ILP techniques (static & dynamic)
* Branch prediction
* Superscalar and out-of-order execution

---

## 🧪 Programming Focus

* All **ILP-level codes** will be written in **RISC-V assembly**
* Programs will be **assembled and simulated using RARS**
* Emphasis on:

  * Pipeline behavior
  * Instruction scheduling
  * Hazard identification and mitigation
  * Performance reasoning

---

## 🛠 Toolchain: RARS (RISC-V Assembler and Runtime Simulator)

### 🔽 Download RARS

Download the official RARS JAR file from:

```
https://github.com/TheThirdOne/rars/releases/download/v1.6/rars1_6.jar
```

---

### ▶ Running RARS

#### Prerequisites

* Java **JDK 8 or later** must be installed
  Check using:

```bash
java -version
```

#### Run RARS (GUI mode)

```bash
java -jar rars1_6.jar
```

#### Run RARS (Command-line mode)

```bash
java -jar rars1_6.jar nc program.s
```

* `nc` → no GUI, command-line execution
* `program.s` → your RISC-V assembly file

---

## 🧠 RISC-V Basics (Quick Reference)

### 🔹 RISC-V Overview

* Open, modular **Reduced Instruction Set Computer (RISC)** architecture
* Fixed **32-bit instruction format** (for RV32)
* Load–store architecture
* Designed for extensibility and research

---

### 🔹 Register File (RV32I)

| Register | Name   | Purpose            |
| -------- | ------ | ------------------ |
| x0       | zero   | Always 0           |
| x1       | ra     | Return address     |
| x2       | sp     | Stack pointer      |
| x5–x7    | t0–t2  | Temporaries        |
| x8–x9    | s0–s1  | Saved registers    |
| x10–x17  | a0–a7  | Function arguments |
| x18–x27  | s2–s11 | Saved registers    |
| x28–x31  | t3–t6  | Temporaries        |

---

### 🔹 Instruction Types

* **R-type** → Register-register operations
  `add, sub, and, or`
* **I-type** → Immediate & loads
  `addi, lw`
* **S-type** → Stores
  `sw`
* **B-type** → Branches
  `beq, bne`
* **U-type** → Upper immediates
  `lui, auipc`
* **J-type** → Jumps
  `jal`

---

### 🔹 Common RISC-V Extensions

| Extension | Meaning                         |
| --------- | ------------------------------- |
| **I**     | Integer base (mandatory)        |
| **M**     | Integer multiplication/division |
| **A**     | Atomic instructions             |
| **F**     | Single-precision floating point |
| **D**     | Double-precision floating point |
| **C**     | Compressed instructions         |

> In this course, we primarily use **RV32I** and **RV32IM**.

---

### 🔹 Data Types (RV32)

| Type      | Size    |
| --------- | ------- |
| Byte      | 8 bits  |
| Half-word | 16 bits |
| Word      | 32 bits |

Memory is **byte-addressable** and **little-endian**.



## 📌 Notes for Students

* Follow the **textbook rigorously**—it is central to this course.
* Always **justify performance claims quantitatively**.
* Comment your RISC-V code clearly, especially when demonstrating ILP.
* Use RARS pipeline and register views for debugging and analysis.
* **Readme will be extended later on**

---

Happy learning and exploring modern computer architecture 🚀

