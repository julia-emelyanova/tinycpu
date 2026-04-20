# Part 1 — ALU

This directory contains the **Logisim Evolution circuits for the ALU** described in the first article of the series *Overengineering a Factorial* - [Overengineering a Factorial. Part 1, The ALU.](https://julia-em.dev/notes/cpu-factorial-part-1-alu/)

The goal of this stage is to build a minimal **Arithmetic Logic Unit** capable of performing several logical and arithmetic operations. This component will later become the computational core of the processor developed in the series.

The circuits here demonstrate:

- basic logic gates
- multiplexers
- a simple ALU controlled by operation codes

You can read the full explanation in the article:

→ [Overengineering a Factorial. Part 1, The ALU.](https://julia-em.dev/notes/cpu-factorial-part-1-alu/)

### ALU Specification v0.1


**Inputs**:
  A[16:0], B[16:0], 
  F[3:0]

**Outputs**:
  Result[16:0]

**Operations:**
| OpCode| ALU function |
|-----|--------------|
| 000 | A AND B      |
| 001 | A OR B       |
| 010 | A+B          |
| 011 | Not used     |
| 100 | A AND NOT(B) |
| 101 | A OR NOT(B)  |
| 110 | A - B        |
| 111 | SLT          |

**Latency:**
  1 cycle

## Circuit Preview

The ALU subcircuit:

![ALU circuit](images/alu.png)

The main circuit:

![Main circuit](images/main.png)

## Series

This circuit is part of the project:

→ [Overengineering a Factorial](https://julia-em.dev/notes/cpu-factorial/)