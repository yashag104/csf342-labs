# CS F342 – Computer Architecture Lab 7

## Objective
Extend the processor to support control flow, integrate into a single-cycle design, and transition toward pipelined execution with hazard analysis and mitigation.

---

## Task 1: Add Control Instructions

### Instructions to implement
- beq rs1, rs2, imm
- bne rs1, rs2, imm
- jal rd, imm

### Expected design additions
- Branch comparator (rs1 == rs2, rs1 != rs2)
- PC update logic:
  - PC + 4 (default)
  - PC + imm (branch/jump target)
- Control signals:
  - Branch
  - Jump
  - PCSrc

### Notes
For jal:
- rd ← PC + 4
- PC ← PC + imm

---

## Task 2: Single-Cycle Integration

### Requirements
- Keep control logic behavioral (use the `switch` statement)
- Integrate:
  - ALU
  - Register file
  - Data memory
  - Instruction memory
  - Immediate generator
  - Control unit

### Deliverables
- Top-level module named `cpu_SC.dut`

---

## Task 3: Run Example Programs
Assemble these using the online assembler to convert to machine code.  
Manually load the machine code into the IMEM in your testbench.  
Execute and note outputs.  
Complete the table after each program.  
Write the program test benches so that they will output the table below automatically.

### Program 1 (test bench name `tb1.v`)

```
addi x1, x0, 10
addi x2, x0, 20
addi x4, x0, 5
xori x3, x1, 0xFF
addi x3, x3, 1
sub  x5, x4, x1
add  x6, x2, x4
add  x7, x3, x2
```

| Register | Expected Value | Observed Value |
|----------|--------------|----------------|
| x1 | 10  | 10  |
| x2 | 20 | 20  |
| x3 | -10 | 246  |
| x4 | 5 | 5  |
| x5 | -5 | -5  |
| x6 | 25  | 25  |
| x7 | 10 | 266  |

---

### Program 2 (test bench name `tb2.v`)

```
addi x1, x0, 10
addi x2, x0, 5
add  x3, x1, x2
sub  x4, x3, x2
lw   x5, 0(x3)
add  x6, x5, x1
sw   x6, 0(x2)
```

| Register | Expected Value | Observed Value |
|----------|--------------|----------------|
| x1 | 10  | 10  |
| x2 | 5 | 5  |
| x3 | 15  | 15  |
| x4 | 10  | 10  |
| x5 | 0  |  0 |
| x6 | 10 | 10  |
| x7 | 0 | 0  |

---

### Program 3: Fibonacci (test bench name `tb3.v`)

```
addi x1, x0, 0
addi x2, x0, 1
addi x3, x0, 10

loop:
add  x4, x1, x2
add  x1, x2, x0
add  x2, x4, x0
addi x3, x3, -1
bne  x3, x0, loop
```

| Register | Expected Value | Observed Value |
|----------|--------------|--------------|
| x1 | 55 | 55 |
| x2 | 89 | 89 |
| x3 | 0 | 0 |
| x4 | 89 | 89 |

---

## Task 4: Add Pipeline Registers

### IF/ID
- Instruction
- PC

### ID/EX
- PC
- rs1_val, rs2_val
- Immediate
- Destination register
- Control signals:
  - ALUOp
  - ALUSrc
  - MemRead
  - MemWrite
  - RegWrite
  - MemToReg

### EX/MEM
- ALU result
- Store data
- Destination register
- Control signals:
  - MemRead
  - MemWrite
  - RegWrite
  - MemToReg

### MEM/WB
- Memory data
- ALU result
- Destination register
- Control signals:
  - RegWrite
  - MemToReg
  

### Deliverables
- Top-level module named `cpu_pip.dut`

---

## Task 5: Run Programs on Pipelined version
Use the same test benches that you wrote above. Only run with the new cpu module.  
Note the expected and actual outcomes and comment on them.

- The pipelined CPU currently lacks hazard detection, data forwarding, or control flush logic.
- **Data Hazards (RAW):** Results are written to the register file at the end of the Write-Back stage, meaning an instruction attempting to read a recently calculated register within the next 3 cycles will fetch the stale value instead. This is why `x3`, `x5`, and `x7` in Program 1 produced wrong values, and why almost all dependent registers in Programs 2 and 3 failed outright (giving `0`).
- **Control Hazards:** The branch target in Program 3 is evaluated in the EX stage, meaning the CPU continues fetching the two next sequential instructions into the pipeline before the Program Counter updates. Since there is no hardware flush, these incorrect instructions execute, further corrupting the state.

---

## Task 6: Fix the pipelined CPU
Modify the **assembly programs only without changing the hardware** to fix the issues you observe above.

Program 1:
```
addi x1, x0, 10
addi x2, x0, 20
addi x4, x0, 5
nop
xori x3, x1, 0xFF
nop
nop
nop
addi x3, x3, 1
sub  x5, x4, x1
add  x6, x2, x4
nop
add  x7, x3, x2
```

Program 2:
```
addi x1, x0, 10
addi x2, x0, 5
nop
nop
nop
add  x3, x1, x2
nop
nop
nop
sub  x4, x3, x2
lw   x5, 0(x3)
nop
nop
nop
add  x6, x5, x1
nop
nop
nop
sw   x6, 0(x2)

```

Program 3:
```
addi x1, x0, 0
addi x2, x0, 1
addi x3, x0, 10

loop:
add  x4, x1, x2
add  x1, x2, x0
nop
nop
add  x2, x4, x0
addi x3, x3, -1
nop
nop
nop
bne  x3, x0, loop
nop
nop

```

### Outputs
```
========== Task 5 — Program 1 ==========
  Reg   Expected         Observed       
  PASS  x1   = 10
  PASS  x2   = 20
  FAIL  x3   : expected -10, got 246
  PASS  x4   = 5
  PASS  x5   = -5
  PASS  x6   = 25
  FAIL  x7   : expected 10, got 266
=========================================

========== Task 5 — Program 2 ==========
  Reg   Expected         Observed       
  PASS  x1   = 10
  PASS  x2   = 5
  PASS  x3   = 15
  PASS  x4   = 10
  PASS  x5   = 0
  PASS  x6   = 10
  PASS  x7   = 0
=========================================

========== Task 5 — Program 3 (Fibonacci) ==========
  Reg   Expected         Observed       
  FAIL  x1   : expected 55, got 0
  FAIL  x2   : expected 89, got 0
  PASS  x3   = 0
  FAIL  x4   : expected 89, got 0
=====================================================


```
