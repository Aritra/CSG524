## **Lab Sheet 4: Memory-Mapped I/O, Keyboard, and Bitmap Graphics in RISC-V (RARS)**

---

## **Objective**

After completing this lab, students will:

* Understand **bare-metal I/O** without OS abstractions
* Use **memory-mapped I/O (MMIO)**
* Poll hardware registers correctly
* Interact with **keyboard and display**
* Write to **framebuffer memory**
* Implement simple **graphics primitives**
* Build an **interactive real-time program**

---

# **Part 0 — Working Principle of I/O on Bare-Metal Processors**

---

## ❖ How does a CPU talk to devices?

Unlike syscalls (OS-level), **bare-metal hardware uses memory-mapped registers**.

Devices appear as **special memory addresses**.

When you:

```
lw/sw/lb/sb
```

on those addresses → you are **talking to hardware**, not RAM.

---

## ❖ Memory-Mapped I/O Model

```
CPU  <---->  Memory Bus  <---->  Device Registers
```

Each device exposes:

| Type             | Meaning             |
| ---------------- | ------------------- |
| Control register | status/ready flags  |
| Data register    | actual input/output |

---

## ❖ Polling Flow

### Keyboard read

```
loop:
    check ready bit
    if ready → read data
```

### Display write

```
loop:
    check ready bit
    if ready → write data
```

This is called **busy waiting (polling)**.

---

## ❖ Why not interrupts?

Real systems use:

* interrupts
* DMA
* drivers

But RARS uses **polling** to keep things simple.

---

---

# **Part 1 — Keyboard + Display MMIO Simulator**

---

## **1.1 Launching the Tool**

1. Run:

```bash
java -jar rars.jar
```

2. Tools → **Keyboard and Display MMIO Simulator**

---

## **1.2 MMIO Addresses**

| Purpose          | Address    | Meaning        |
| ---------------- | ---------- | -------------- |
| Keyboard Control | 0xFFFF0000 | 1 if key ready |
| Keyboard Data    | 0xFFFF0004 | ASCII code     |
| Display Control  | 0xFFFF0008 | 1 if ready     |
| Display Data     | 0xFFFF000C | write ASCII    |

All operations are **1 byte**.

---

---

## **1.3 Basic Echo Program (Working)**

```assembly
.data
msg: .asciz "Type (ENTER to quit):\n"

.text
.globl main
main:

# print prompt
la a0, msg
li a7, 4
ecall

li t0, 13          # ENTER ASCII

read_loop:

# wait keyboard ready
li t1, 0xFFFF0000
wait_k:
lb t2, 0(t1)
beqz t2, wait_k

# read char
li t1, 0xFFFF0004
lb t3, 0(t1)
beq t3, t0, exit

# wait display ready
li t1, 0xFFFF0008
wait_d:
lb t2, 0(t1)
beqz t2, wait_d

# print char
li t1, 0xFFFF000C
sb t3, 0(t1)

j read_loop

exit:
li a7, 10
ecall
```

---

---

# **Part 2 — Bitmap Display (Framebuffer Programming)**

---

## **2.1 Framebuffer Concept**

Bitmap display behaves like:

```
uint32 framebuffer[WIDTH * HEIGHT];
```

Each pixel:

```
0xAARRGGBB
```

---

## **2.2 Pixel Address Formula**

For pixel at $$(x,y)$$:

$$
\text{addr} = \text{base} + 4 \times (y \times \text{width} + x)
$$

---

## **2.3 Drawing One Pixel**

```assembly
li t0, 0x10010000   # base
mul t1, y, width
add t1, t1, x
slli t1, t1, 2
add t0, t0, t1
sw color, 0(t0)
```

---

---

# **Part 3 — Drawing Primitives**

---

## Example: Drawing a Horizontal Line in Hardcoded Red (full working code)

```assembly
.data
WIDTH: .word 512           # Display width (matches Bitmap Display setting)
RED:   .word 0xFFFF0000    # Fully opaque red (0xAARRGGBB)
.text
.globl main
main:
    # Set starting position and length
    li t0, 5             # start_x
    li t1, 15              # start_y
    li t2, 40             # length

    la t3, WIDTH          # Load address of WIDTH
    lw t3, 0(t3)          # Load width into t3
    add t4, t0, t2        # end_x = start_x + length
    bgt t4, t3, line_too_long  # If line would cross right edge, abort

    la t5, RED            # Load address of RED
    lw t5, 0(t5)          # t5 = RED value

    li t6, 0              # i = 0 (loop counter)

line_loop:
    bge t6, t2, done_line
    mul a0, t1, t3        # a0 = y * width
    add a0, a0, t0        # a0 = y * width + start_x
    add a0, a0, t6        # a0 = y * width + start_x + i
    slli a0, a0, 2        # byte offset = (y * width + start_x + i) * 4
    li t5, 0x10010000     # Bitmap base address (overwrite t5 safely after RED used)
    add t5, t5, a0        # t5 = address of current pixel
    lw a0, RED            # reload RED value for sw
    sw a0, 0(t5)          # store red pixel at computed address
    addi t6, t6, 1        # i++
    j line_loop

line_too_long:
    li a7, 10             # Exit (line would exceed bounds)
    ecall

done_line:
    li a7, 10             # Exit
    ecall
```


---

## Example: Draw Rectangle (filled, function only)

```assembly
# a0=x, a1=y, a2=w, a3=h, a4=color
draw_rect:

li t6, 0x10010000

li t0, 0
row_loop:
bge t0, a3, done

li t1, 0
col_loop:
bge t1, a2, next_row

mul t2, a1, s0      # s0 holds width
add t2, t2, a0
add t2, t2, t1
add t2, t2, t0
slli t2, t2, 2
add t3, t6, t2
sw a4, 0(t3)

addi t1, t1, 1
j col_loop

next_row:
addi t0, t0, 1
j row_loop

done:
ret
```

---

---

# **Part 4 — Interactive Rectangle Movement (Keyboard + Graphics)**

---

## Goal

Move a rectangle **1 pixel** using keys:

| Key | Action |
| --- | ------ |
| W   | up     |
| A   | left   |
| S   | down   |
| D   | right  |

---

## ✅ Working Sample Program

```assembly
.data
WIDTH:  .word 256
HEIGHT: .word 256
COLOR:  .word 0xFFFF0000

.text
.globl main
main:

lw s0, WIDTH      # width
li s1, 100        # x
li s2, 100        # y

loop:

# ---- wait key ----
li t0, 0xFFFF0000
wait_k:
lb t1, 0(t0)
beqz t1, wait_k

li t0, 0xFFFF0004
lb t2, 0(t0)

# ---- clear screen (simple redraw logic) ----
jal clear_screen

# movement
li t3, 'w'
beq t2, t3, up

li t3, 's'
beq t2, t3, down

li t3, 'a'
beq t2, t3, left

li t3, 'd'
beq t2, t3, right
j draw

up:   addi s2, s2, -1 ; j draw
down: addi s2, s2, 1  ; j draw
left: addi s1, s1, -1 ; j draw
right:addi s1, s1, 1

draw:
mv a0, s1
mv a1, s2
li a2, 10
li a3, 10
lw a4, COLOR
jal draw_rect

j loop
```

*(Use earlier `draw_rect` and implement a simple screen clear loop.)*

---

---

# **Student Tasks**

---

## **Task 1 — Enhanced Keyboard Tool**

Modify echo program to:

* track word count
* support backspace
* update count dynamically

---

## **Task 2 — Serial/Keyboard Graphics Control**

Write a complete program that:

### Requirements

* Uses keyboard MMIO
* Uses bitmap framebuffer
* Draws a rectangle
* Moves rectangle using WASD
* Movement step = 1 pixel
* No syscalls allowed inside loop (pure MMIO only)

---

---

# **Expected Learning Outcomes**

Students should now:

* Understand bare-metal device control
* Use MMIO safely
* Perform framebuffer math
* Build interactive programs
* Appreciate how OS hides hardware complexity

---

