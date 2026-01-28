
## **Lab Sheet 2: Floating Point Representation and Transcendental Functions**

---

## **Objective**

The objective of this lab is to:

* Understand IEEE 754-2008 floating point representation
* Learn how floating point numbers are handled in RISC-V
* Use single and double precision operations in RARS
* Implement transcendental functions using series expansion
* Analyze numerical efficiency and approximation techniques

---

## **1. IEEE 754-2008 Floating Point Standard**

### Supported Formats

IEEE 754-2008 defines multiple floating point formats. The most relevant ones are:

| Format | Name                | Total Bits |
| ------ | ------------------- | ---------- |
| H      | Half Precision      | 16         |
| F      | Single Precision    | 32         |
| D      | Double Precision    | 64         |
| Q      | Quadruple Precision | 128        |

In this lab, we focus on **Single (F)** and **Double (D)** precision, as supported by RARS.

---

## **2. Why Is It Called “Floating Point”?**

A floating point number is represented as:

$$
(-1)^s \times m \times b^e
$$

Where:

* $$s$$ is the sign bit
* $$m$$ is the significand (mantissa)
* $$b$$ is the base (2 for binary)
* $$e$$ is the exponent

The **decimal (binary) point “floats”** depending on the exponent value, allowing representation of very large and very small numbers.

---

## **3. Single and Double Precision Representation**

### **Single Precision (32-bit)**

| Field    | Bits |
| -------- | ---- |
| Sign     | 1    |
| Exponent | 8    |
| Mantissa | 23   |

Exponent bias = **127**

Value represented:

$$
(-1)^s \times (1.m) \times 2^{(e - 127)}
$$

---

### **Double Precision (64-bit)**

| Field    | Bits |
| -------- | ---- |
| Sign     | 1    |
| Exponent | 11   |
| Mantissa | 52   |

Exponent bias = **1023**

Value represented:

$$
(-1)^s \times (1.m) \times 2^{(e - 1023)}
$$

---

## **4. Range of Floating Point Numbers**

### **Single Precision Range**

* Smallest positive normalized number:

$$
2^{-126}
$$

* Largest finite number:

$$
(2 - 2^{-23}) \times 2^{127}
$$

Approximate range:

$$
10^{-38} \text{ to } 10^{38}
$$

---

### **Double Precision Range**

* Smallest positive normalized number:

$$
2^{-1022}
$$

* Largest finite number:

$$
(2 - 2^{-52}) \times 2^{1023}
$$

Approximate range:

$$
10^{-308} \text{ to } 10^{308}
$$

---

## **5. Floating Point Instructions in RARS**

RARS supports **single (`.s`)** and **double (`.d`)** precision instructions.

### **Arithmetic Instructions**

| Operation | Single   | Double   | Explanation             |
| --------- | -------- | -------- | ----------------------- |
| Add       | `fadd.s` | `fadd.d` | Floating addition       |
| Sub       | `fsub.s` | `fsub.d` | Floating subtraction    |
| Mul       | `fmul.s` | `fmul.d` | Floating multiplication |
| Div       | `fdiv.s` | `fdiv.d` | Floating division       |

---

### **Conversion Instructions**

| Instruction | Meaning         |
| ----------- | --------------- |
| `fcvt.s.w`  | int → single    |
| `fcvt.d.w`  | int → double    |
| `fcvt.w.s`  | single → int    |
| `fcvt.w.d`  | double → int    |
| `fcvt.d.s`  | single → double |
| `fcvt.s.d`  | double → single |

---

### **Load / Store**

| Instruction | Meaning      |
| ----------- | ------------ |
| `flw`       | load single  |
| `fsw`       | store single |
| `fld`       | load double  |
| `fsd`       | store double |

---

## **6. Computing sin(x) and cos(x) Using Maclaurin Series**

### **Maclaurin Series**

$$
\sin(x) = x - \frac{x^3}{3!} + \frac{x^5}{5!} - \cdots
$$

$$
\cos(x) = 1 - \frac{x^2}{2!} + \frac{x^4}{4!} - \cdots
$$

> **Important:**
> Input $$x$$ is given in **degrees**. Convert it to radians using:
>
> $$
> x_{\text{rad}} = x \times \frac{\pi}{180}
> $$

---

## **7. Sample RARS Code: sin(x) and cos(x)**

```assembly
.data
pi:     .double 3.141592653589793
deg:    .double 30.0

.text
.globl main
main:
    # Load address of deg and read value
    la t0, deg
    fld f0, 0(t0)          # f0 = degree value

    # Load address of pi and read value
    la t1, pi
    fld f1, 0(t1)          # f1 = pi

    # Convert degrees to radians: x = x * pi / 180
    fmul.d f0, f0, f1

    li t2, 180
    fcvt.d.w f2, t2
    fdiv.d f0, f0, f2      # f0 = radians

    # Maclaurin sin(x): x - x^3 / 3!
    fmul.d f3, f0, f0      # x^2
    fmul.d f4, f3, f0      # x^3

    li t3, 6               # 3!
    fcvt.d.w f5, t3
    fdiv.d f4, f4, f5      # x^3 / 6

    fsub.d f10, f0, f4     # sin(x) approximation

    # Print result
    fmv.d fa0, f10
    li a7, 3               # print double
    ecall

    # Exit
    li a7, 10
    ecall
```

> **Note:**
> The same structure can be extended for `cos(x)` by changing the series.

---

## **Student Task 1**

### **Part A: Improving sin(x) Approximation**

Use the periodicity of sine:

$$
\sin(x) = \sin(x \bmod 2\pi)
$$

Reduce the input angle $$x$$ to a smaller equivalent angle before computing the series.

**Goal:**

* Fewer Maclaurin terms
* Faster convergence
* Better numerical stability

---

### **Part B: Fast Approximation of $$e^x$$**

#### **Mathematical Breakdown**

For $$x > 1$$, write:

$$
x = n + f
$$

Where:

* $$n$$ is the integer part
* $$f$$ is the fractional part, $$0 \le f < 1$$

Then:

$$
e^x = e^{n+f} = e^n \cdot e^f
$$

Using:

$$
e^n = 2^{n / \ln(2)}
$$

Approximate $$e^f$$ using Taylor expansion:

$$
e^f = 1 + f + \frac{f^2}{2!} + \frac{f^3}{3!} + \cdots
$$

---

#### **Algorithm: Binary Exponentiation Inspired Method**

1. Input $$x$$
2. Compute:

   * $$n = \lfloor x \rfloor$$
   * $$f = x - n$$
3. Approximate $$e^f$$ using Taylor series
4. Compute $$2^n$$ using repeated multiplication
5. Multiply both parts to get $$e^x$$

---

## **Student Task 2**

### **Implement $$e^x$$ in RISC-V**

Write a RISC-V program that:

1. Computes $$e^x$$ for $$x > 0$$
2. Uses:

   * Taylor expansion
   * Efficient register usage
3. Hardcode the input $$x$$
4. Optimize for:

   * Fewer floating point operations
   * Minimal register pressure

---

## **Submission Guidelines**

* Use **double precision** (`.d`) unless specified otherwise
* Comment every mathematical step
* Clearly mention approximation order
* Ensure the program runs correctly in **RARS**

---

## **Expected Learning Outcome**

After completing this lab, students should be able to:

* Understand floating point representation and limitations
* Use RISC-V floating point instructions effectively
* Implement numerical approximations
* Reason about accuracy vs performance trade-offs

---

**End of Lab Sheet 2**
