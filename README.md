# 8-bit Microprocessor — Array Rotation MCU
### Built in Logisim Evolution

A fully functional custom 8-bit microprocessor designed and implemented in Logisim Evolution. The processor executes a **cyclic left-rotation** on an integer array `a[10]` continuously in hardware.

---

## Target Program

The processor executes the following C code in hardware:

```c
int a[10] = { 0,1,2,3,4,5,6,7,8,9 };
int i = 0, j = 0;

void main() {
    while(1) {
        j = a[0];
        for(i = 0; i < 9; i++)
            a[i] = a[i+1];
        a[9] = j;
    }
}
```

Each iteration performs a **cyclic left rotation**:
```
{0,1,2,3,4,5,6,7,8,9} → {1,2,3,4,5,6,7,8,9,0} → {2,3,4,5,6,7,8,9,0,1} → ...
```

---

## Processor Architecture

The MCU is built from 5 sub-circuits connected in the Main canvas:

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Counter   │────▶│  Program    │ ────▶│     IR      │
│    (PC)     │      │    ROM      │      │   Register  │
└─────────────┘      └─────────────┘      └──────┬──────┘
                                               │
                    ┌──────────────────────────▼──────┐
                    │         Splitter (16-bit)       │
                    │  15-12:opcode  11-9:RD  8-6:RS1 │
                    │  5-3:RS2       2-0:IMM          │
                    └───┬──────────┬──────────┬───────┘
                        │          │          │
               ┌────────▼──┐  ┌───▼───┐  ┌──▼────────┐
               │  Control  │  │  Reg  │  │    ALU    │
               │   Unit    │  │  File │  │  8-bit    │
               └────────┬──┘  └───┬───┘  └──┬────────┘
                        │         │         │
                        └───────▶└────────▶
                                            │
                                    ┌───────▼────────┐
                                    │   Data Memory  │
                                    │   RAM (Array)  │
                                    └────────────────┘
```

### Sub-circuits

| Sub-circuit | Components | Purpose |
|---|---|---|
| **ProgramCounter** | 8-bit - Register, Adder, MUX | Holds current instruction address, increments each cycle or loads jump target |
| **ProgramROM** | ROM 16×16-bit | Stores the 17-instruction machine code program |
| **RegisterFile** | 8×Register, Decoder, 2×MUX | Eight 8-bit general purpose registers R0–R7 with dual read ports |
| **ALU** | Adder, Subtractor, AND, OR, NOT, MUX, D Flip-Flop | 8-bit arithmetic and logic, produces RESULT and NEG flag |
| **ControlUnit** | Splitter, AND/OR/NOT gates, MUX | Decodes 4-bit opcode and drives all control signals |
| **DataMemory** | RAM 16×8-bit | Stores array a[0..9] at addresses 0–9 |

---

## Instruction Set Architecture (ISA)

### Instruction Format — 16-bit fixed width

```
 15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│      OPCODE   │      RD       │      RS1      │  RS2  │  IMM  │
│     (4 bits)  │    (3 bits)   │    (3 bits)   │(3 bits│(3 bits│
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  bits 15-12          bits 11-9       bits 8-6      bits 5-3  bits 2-0
```

### Opcode Table

| Opcode | Hex | Mnemonic | Operation |
|---|---|---|---|
| 0001 | 1 | `LOAD Rd, [RS1]` | Rd = RAM[RS1] |
| 0010 | 2 | `STORE [RS1], RS2` | RAM[RS1] = RS2 |
| 0011 | 3 | `MOVE Rd, RS1` | Rd = RS1 |
| 0100 | 4 | `ADD Rd, RS1, RS2` | Rd = RS1 + RS2 |
| 0101 | 5 | `SUB Rd, RS1, RS2` | Rd = RS1 - RS2 |
| 0110 | 6 | `CMP RS1, RS2` | RS1 - RS2, set NEG flag only |
| 0111 | 7 | `JMP addr` | PC = lower 8 bits of instruction |
| 1000 | 8 | `JLT addr` | if NEG=1 then PC = addr |
| 1001 | 9 | `LOADX Rd, [RS1], #IMM` | Rd = RAM[RS1+IMM] |
| 1010 | A | `STOREX [RS1+IMM], RS2` | RAM[RS1 + IMM] = RS2 |
| 1011 | B | `MOVEC Rd, #IMM` | Rd = IMM (3-bit immediate) |

### Register File

| Register | Code | Role in Program |
|---|---|---|
| R0 | 000 | Constant 0 — never written |
| R1 | 001 | j — saves a[0] before rotation |
| R2 | 010 | i — loop counter (0 to 8) |
| R3 | 011 | Scratch — current index i |
| R4 | 100 | Scratch — a[i+1] value |
| R5 | 101 | Constant 9 — loop limit |
| R6 | 110 | Constant 1 — loop increment |
| R7 | 111 | Unused |

---

## Machine Code Program

### ROM Contents
```
v2.0 raw
0000 1200 BA07 B602 4B58 BC01 4400 60A8
800A 7010 4610 48F0 1900 20E0 44B0 7007
2148 7001
```

### Disassembly

| Addr | Hex | Assembly | Effect |
|---|---|---|---|
| 00 | 0000 |---|---|
| 01 | 1200 | LOAD R1, [R0] | R1 = j = RAM[0] = a[0] |
| 02 | BA07 | MOVEC R5, #7 | R5 = 7 |
| 03 | B602 | MOVEC R3, #2 | R3 = 2 (temp) |
| 04 | 4B58 | ADD R5, R5, R3 | R5 = 9 (loop limit) |
| 05 | BC01 | MOVEC R6, #1 | R6 = 1 (increment) |
| 06 | 4400 | ADD R2, R0, R0 | R2 = i = 0 |
| 07 | 60A8 | CMP R2, R5 | i - 9, NEG=1 if i<9 |
| 08 | 800A | JLT 0x0A | if i<9 jump to loop body |
| 09 | 7010 | JMP 0x10 | i>=9 exit loop |
| 0A | 4610 | ADD R3, R0, R2 | R3 = i |
| 0B | 48F0 | ADD R4, R3, R6 | R4 = i+1 |
| 0C | 1900 | LOAD R4, [R4] | R4 = RAM[i+1] = a[i+1] |
| 0D | 20E0 | STORE [R3], R4 | RAM[i] = a[i+1] |
| 0E | 44B0 | ADD R2, R2, R6 | i++ |
| 0F | 7007 | JMP 0x07 | back to CMP |
| 10 | 2148 | STORE [R5], R1 | RAM[9] = j = old a[0] |
| 11 | 7001 | JMP 0x01 | while(1) restart |

### RAM Initial Contents
```
v2.0 raw
00 01 02 03 04 05 06 07 08 09
```
Addresses 0–9 hold `a[0]` through `a[9]`.

---

## Control Signals

| Signal | Goes to | Meaning when HIGH |
|---|---|---|
| RegWrite | RegisterFile WR_EN | Write result into destination register |
| MemWrite | DataMemory str | Write data into RAM |
| MemToReg | MUX before WR_DATA | Send RAM output to register (not ALU) |
| JUMP_SEL | PC JUMP_SEL pin | Load jump address instead of counting |
| AluSrc | MUX before ALU B | Use IMM as B input instead of RS2 register |
| OP [2:0] | ALU OP input | Select ALU operation |

---

## How to Run

### Requirements
- [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution) v3.x or later
- Java 11 or later

### Steps
1. Clone this repository
2. Open `mcu_array_rotate.circ` in Logisim Evolution
3. Go to `Simulate → Tick Frequency` → set to **4 Hz** (slow, for observation)
4. Right-click the ROM → **Edit Contents** → paste the ROM contents above
5. Right-click the RAM → **Edit Contents** → paste the RAM contents above
6. Press the **RESET** button on the canvas
7. Go to `Simulate → Go` (or press `Ctrl+K`)
8. Watch the RAM contents rotate every ~17 clock cycles

---

---

## Authors

**Prince Yadav**

Department of Electrical Engineering

---
