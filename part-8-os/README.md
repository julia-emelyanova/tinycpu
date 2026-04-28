# Custom Minimal CPU Specification

## 1. Overview

This project implements a minimal custom CPU designed as part of the *(Overengineering a Factorial)[https://julia-em.dev/notes/cpu-factorial/]* series.

The original goal was to compute a factorial, but the project gradually evolved into a complete small computing system: a processor, an instruction set, an assembler, memory layout, memory-mapped I/O, function calls, and a minimal single-threaded operating system capable of loading and running user programs.

The CPU is intentionally simple. It is not meant to be efficient or realistic in every detail. Its purpose is to make the full execution path understandable: from assembly code, to machine code, to hardware-level execution.

# 2. Hardware Architecture

## Main components

The processor consists of the following hardware blocks:

* Program Counter (`PC`)
* Instruction Register
* Control Unit
* ALU
* General-purpose registers
* Stack Pointer (`SP`)
* ROM (for storing code)
* RAM (for storing temporary variables and state)
* Memory-mapped I/O
* Keyboard input interface
* Text output interface

## Program Counter

The `PC` stores the address of the currently executing instruction in ROM.

After most instructions, `PC` is incremented to point to the next instruction.

Jump and call instructions modify `PC` directly.

## Registers

The processor currently uses a small register set:

| Register | Purpose                            |
| -------- | ---------------------------------- |
| `A`      | Main general-purpose register      |
| `B`      | Secondary general-purpose register |
| `PC`     | Program Counter                    |
| `SP`     | Stack Pointer                      |

Registers `A` and `B` are used for arithmetic, logic operations, comparisons, memory access, and passing simple values between routines.

`SP` is used to implement function calls and returns.

# 3. ALU

The ALU operates mainly on registers `A` and `B`.

It supports several types of operations:

- Logical: AND, OR
- Arithmetic: Addition, Subtraction
- Comparison: SLT (Set on Less Than) — outputs 1 if A < B, otherwise 0. (Used in conditional branches and comparisons.)

The operation performed by the ALU is determined by a 3-bit control signal.

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

The result is stored in register `A`.

# 4. Instruction Format

Each instruction has a fixed-width format:

```text
opcode: 5 bits
operand: 16 bits
padding: to 24 bits total
```

The final machine instruction is encoded as a 6-digit big-endian hexadecimal value.

Example structure:

```text
[ opcode ][ operand ]
5 bits     16 bits
```

The assembler is responsible for:

* expanding macros
* resolving labels
* allocating variables
* resolving symbolic addresses
* encoding instructions into HEX

# 5. Instruction Set Architecture

## Instruction list

| Mnemonic    |  Opcode | Description                                                            |
| ----------- | ------: | ---------------------------------------------------------------------- |
| `NOOP`      | `00000` | Do nothing                                                             |
| `LOADIA`    | `00001` | Load immediate value into register A                                   |
| `LOADIB`    | `00010` | Load immediate value into register B                                   |
| `AND`       | `00011` | `A = A AND B`                                                          |
| `OR`        | `00100` | `A = A OR B`                                                           |
| `ADD`       | `00101` | `A = A + B`                                                            |
| `ANDN`      | `00110` | `A = A AND NOT B`                                                      |
| `ORN`       | `00111` | `A = A OR NOT B`                                                       |
| `SUB`       | `01000` | `A = A - B`                                                            |
| `SLT`       | `01001` | Set A based on comparison                                              |
| `JZ`        | `01010` | Jump if A is zero                                                      |
| `JZB`       | `01011` | Jump if B is zero                                                      |
| `STOREA`    | `01100` | Store register A to memory                                             |
| `STOREB`    | `01101` | Store register B to memory                                             |
| `LOADA`     | `01110` | Load memory value into A                                               |
| `LOADB`     | `01111` | Load memory value into B                                               |
| `LOADAIND`  | `10000` | Load into A using address stored in memory/register-indirect mechanism |
| `STOREAIND` | `10001` | Store A using indirect addressing                                      |
| `CALL`      | `10010` | Call function at address                                               |
| `RET`       | `10011` | Return from function                                                   |
| `CALL_REG`  | `10100` | Call function whose address is stored in register A                    |

# 6. Control Flow

## `JZ`

`JZ` jumps to a given ROM address if register `A` is zero.

It is used both for conditional branching and for unconditional jumps by first setting `A` to zero:

```asm
LOADIA 0
JZ SOME_LABEL
```

## `JZB`

`JZB` works similarly, but checks register `B`.

## `CALL`

`CALL` is used to call a function at a known address.

Example:

```asm
CALL FUNC_PRINT_STRING
```

The assembler resolves `FUNC_PRINT_STRING` into a concrete ROM address.

At runtime:

1. The CPU stores the address of the next instruction on the stack.
2. `SP` is incremented.
3. `PC` is set to the function address.

## `RET`

`RET` returns from a function.

At runtime:

1. `SP` is decremented.
2. The return address is read from the stack.
3. `PC` is restored to that address.

## `CALL_REG`

`CALL_REG` calls a function whose address is already stored in register `A`.

This instruction is used by tinyOS to call user programs dynamically.

Example:

```asm
CALL FUNC_FIND_PROG
; register A now contains program address
CALL_REG
```

This makes it possible to select a program by name at runtime.

# 7. Memory Architecture

The system has separate ROM and RAM.

ROM stores executable code.

RAM stores runtime data: stack, variables, program tables, and I/O buffers.


## ROM Layout

```text
0x0000–0x00FF    bootloader
0x0100–0xEFFF    user programs
0xF000–0xFFFF    tinyOS
```

The layout is intentionally simple and fixed.

Each program can be assembled independently by providing its own `START_ADDRESS`.

This wastes some memory, because there may be gaps between programs, but it makes development much easier.

## RAM Layout

```text
0x00–0xBD    stack and heap
0xBE–0xEF    program library
0xF0–0xFF    I/O buffer
```

### Stack

The stack starts at low RAM addresses and grows upward.

It stores return addresses for `CALL` / `RET`.

### Heap / variables

Variables are allocated downward from higher available RAM addresses.

### Program library

The region `0xBE–0xEF` stores the tinyOS program table:

```text
PROGRAM_NAME, LF, PROGRAM_ADDRESS, LF
PROGRAM_NAME, LF, PROGRAM_ADDRESS, LF
...
LF
```

This allows the OS to search for a program by name and retrieve its address.

### I/O buffer

The region `0xF0–0xFF` is used for user input.

It stores up to 16 characters, or fewer if the user presses Enter earlier.

# 8. Memory-Mapped I/O

Input and output are implemented through memory-mapped addresses.

|  Address | Name          | Purpose         |
| -------: | ------------- | --------------- |
| `0x0100` | `$TTY_OUT`    | Text output     |
| `0x0101` | `$KBD_DATA`   | Keyboard data   |
| `0x0102` | `$KBD_STATUS` | Keyboard status |

Example output:

```asm
LOADIA 72
STOREA $TTY_OUT
```

This prints the ASCII character `H`.

# 9. Input / Output Model

The CPU does not have special I/O instructions.

Instead, I/O is performed using normal memory instructions.

This keeps the ISA simpler: the same `LOAD` and `STORE` mechanisms are used for both RAM and I/O devices.

## Output

To print a character:

```asm
LOADIA <ASCII_CODE>
STOREA $TTY_OUT
```

## Input

Keyboard input is read through:

```asm
LOADA $KBD_STATUS
LOADA $KBD_DATA
```

The OS routines abstract this into higher-level functions such as:

* read character
* read string
* print string
* read number

# 10. Assembler

The assembler is AI-assisted and prompt-driven.

It takes assembly code and produces machine code as 6-digit HEX instructions.

The assembler is responsible for:

1. Splitting sections
2. Expanding macros
3. Resolving labels
4. Allocating variables
5. Replacing symbolic addresses
6. Encoding instructions


## Label resolution

Labels are resolved relative to `START_ADDRESS`.

Example:

```asm
START_ADDRESS: 0xF000

_LOOP:
    LOADIA 0
    JZ _LOOP
```

The assembler calculates the concrete ROM address of `_LOOP`.


## Function addresses

Functions may be provided explicitly:

```asm
FUNC_READ_USER_INPUT: 0x040E
FUNC_CREATE_PROGRAM_LIB: 0x0100
FUNC_FIND_PROG: 0x0500
```

This allows programs to call routines located in different ROM regions.

## Macros

Macros are used to reduce repetitive assembly code.

Example:

```asm
PRINTLN 'Hello'
```

is expanded into repeated sequences of:

```asm
LOADIA <ASCII>
STOREA $TTY_OUT
```

for every character.

# 11. tinyOS

The system includes a minimal single-threaded operating system located at:

```text
0xF000–0xFFFF
```

It is not an OS in the modern sense. It has no privilege separation, no scheduler, and no process isolation.

It is closer to a small runtime library plus command loop.

## tinyOS responsibilities

tinyOS provides:

* program selection
* program dispatch
* reusable I/O routines
* program library construction
* user input handling
* string output

## Main OS loop

The OS:

1. Creates a program library in RAM.
2. Greets the user.
3. Prints available programs.
4. Reads a program name.
5. Searches for the program address.
6. Calls the program using `CALL_REG`.
7. Waits for user input.
8. Repeats.

Simplified structure:

```asm
CALL FUNC_CREATE_PROGRAM_LIB
PRINTLN 'Hi, I am tinyOS.'

_CALL_PROG:
    PRINTLN 'Please name a program you want to run.'
    PRINTLN 'Available programs:'
    CALL FUNC_PRINT_PROGRAMS
    CALL FUNC_READ_USER_INPUT
    CALL FUNC_FIND_PROG

    JZ _CALL_PROG_NOT_FOUND

    CALL_REG

_CALL_PROG_END:
    PRINTLN 'Press any key to continue'
    CALL FUNC_PRESS_ANY_KEY
    LOADIA 0
    JZ _CALL_PROG
```

# 12. What Is Deliberately Missing

This processor/system deliberately does not include:

* interrupts
* scheduler
* multitasking
* privilege modes
* syscall mechanism
* memory protection
* full error handling
* dynamic program loading
* advanced addressing modes
* hardware stack frames
* parameters for functions

These omissions are intentional.

The goal is not to build a realistic general-purpose computer, but to understand the minimal set of mechanisms needed to execute programs in a structured way.

# 13. Current User Programs

The system currently supports simple user programs such as:

* `hi`
* `oddoreven`

The planned original target was:

* `Factorial`

However, by the time the processor, assembler, I/O system, memory model, and tinyOS were implemented, computing factorial itself became an exercise in execution rather than design.

## Adding A New Program

The program table is created at runtime and stored in RAM in the following format:

```text
PROGRAM_NAME LF PROGRAM_ADDRESS LF
PROGRAM_NAME LF PROGRAM_ADDRESS LF
...
PROGRAM_NAME LF PROGRAM_ADDRESS LF
LF
```

Both table cells and rows are separated by the `LF` (line feed) character. The final `LF` also marks the end of the table.

To add a new program, edit the file:

```
code/0x0100_programs_prg_lib
```

Add a new line in the following form:

```text
CREATE_PROGRAM_NAME_ROW addressInTheTable programName programAddress
```

where:

* `addressInTheTable` — the RAM address where the new row will be written
* `programName` — the name of the program as a string
* `programAddress` — the ROM address where the program is located

Since the table is terminated by an `LF`, the address of the final `LF` must also be updated accordingly after adding a new entry.


# 14. Design Philosophy

This CPU is intentionally overengineered for its original purpose.

The interesting part is not the factorial calculation itself, but the infrastructure that makes the calculation possible:

* instruction execution
* control flow
* memory access
* function calls
* I/O
* assembly translation
* program dispatch

In that sense, the processor succeeds not because it computes factorial efficiently, but because it exposes the entire path from code to execution.

# 15. Implementation

The hardware part of the processor is implemented in [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution
).

Logisim is used here not as a precise hardware modeling tool, but as a way to make the structure of the processor visible and testable. It allows building the CPU from basic components (gates, multiplexers, registers) and observing how instructions propagate through the system step by step.
