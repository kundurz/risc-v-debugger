# RVDB: A bare metal NIOS-V debugger

**Authors**: Linus Kundur-Zourntos & Jet Chiang

## Overview 
RVDB is a bare-metal debugger that runs directly on the NIOS-V Processor

Important characteristics:
* The debugger and debugee run on the same processor
* Input comes from a PS/2 keyboard
* Debugger output is displayed through the VGA
* It supports breakpoints, stepping, continuing, register inspection, memory inspection, and disassembly
* It can run on physical hardware or CPULator

## Functionality
RVDB has the following major capabilities:
* Software breakpoints
* Single-instruction and multi-instruction stepping
* Continue execution
* General-purpose register inspection
* Memory examination
* RV32 instruction disassembly
* Breakpoitn listing and removal
* Program restart
* VGA command-line interface


## Architecture
```
                          NIOS-V BARE-METAL DEBUGGER
┌──────────────────────────────────────────────────────────────────────────────┐
│                           DE1-SoC / Nios V System                            │
│                                                                              │
│  ┌─────────────┐       ┌──────────────────────────────────────────────────┐  │
│  │ PS/2        │ scans │                 USER INTERFACE                   │  │
│  │ Keyboard    ├──────►│  ┌──────────┐    ┌───────────────┐               │  │
│  └─────────────┘       │  │ UI Loop  ├───►│ Command Parser│               │  │
│                        │  │  ui.c    │◄───┤  commands.c   │               │  │
│  ┌─────────────┐ chars │  └────┬─────┘    └───────┬───────┘               │  │
│  │ VGA Display │◄──────┤       │                  │                       │  │
│  └─────────────┘       └───────┼──────────────────┼───────────────────────┘  │
│                                 │                  │                         │
│                                 │       ┌──────────┴───────────┐             │
│                                 │       │                      │             │
│                           pause │  continue / step       inspect / modify    │
│                                 │       │                      │             │
│                                 ▼       ▼                      ▼             │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                         DEBUGGER ENGINE                                │  │
│  │                                                                        │  │
│  │  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────────┐  │  │
│  │  │ Execution Core  │──►│ Breakpoint      │──►│ Program Memory       │  │  │
│  │  │ core.c          │   │ Manager         │   │                      │  │  │
│  │  │                 │◄──│ breakpoint.c    │◄──│ Original/patched     │  │  │
│  │  │ • continue      │   │                 │   │ instructions         │  │  │
│  │  │ • single-step   │   │ • breakpoint DB │   └──────────┬───────── ─┘  │  │
│  │  │ • trap decision │   │ • JAL patching  │              │              │  │
│  │  └────────┬────────┘   │ • next-PC calc  │    instructions│            │  │
│  │           │            └──────────────────┘              ▼             │  │
│  │           │                                      ┌───────────────┐     │  │
│  │           │              ┌──────────────────┐    │ Disassembler  │     │  │
│  │           │              │ TrapFrame        │    │disassembler.c │     │  │
│  │           │ inspect      │                  │    └───────────────┘     │  │
│  │           └─────────────►│ x1–x31           │                          │  │
│  │                          │ mepc, mcause     │                          │  │
│  │                          │ mstatus, mtval   │                          │  │
│  │                          └────────▲─────────┘                          │  │
│  └───────────────────────────────────┼────────────────────────────────────┘  │
│                                      │ save / restore                        │
│                           ┌──────────┴───────────┐                           │
│                           │ Assembly Trap Handler│                           │
│                           │ trap_handler.S       │                           │
│                           └──────────▲───────────┘                           │
│                                      │ patched JAL                           │
│                                      │ breakpoint / step target              │
│                           ┌──────────┴───────────┐                           │
│                           │   Debugee Program    │                           │
│                           │ program_start → done │                           │
│                           └──────────┬───────────┘                           │
│                                      │                                       │
│                                resume execution                              │
│                                      └───────────────────────────┐           │
│                                                                  ▼           │
│                                                            ┌───────────┐     │
│                                                            │ Nios V CPU│     │
│                                                            └───────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```
## How breakpoints work

The current implementation does not primarily use a conventional `ebreak`. exception. Instead, it replaces an instruction with a `jal trap_handler` instruction:
1. The original instruction is saved in the breakpoint table
2. The instruction is replaced with a jump to `trap_handler`
3. When execution reaches that address, control enters the debugger
4. The assembly handler saves registers into `TrapFrame`
5. The debugger restores the original instruction when necessary.
6. Execution resumes after the trap frame is restored.

Because the NIOS V GDB environment intercepts and manages `ebreak` instructions before the user can handle them, the debugger could not use `ebreak` for its own software breakpoitns. Instead, it replaces instructions with `jal` instructions that transfer control directly to the custom trap handler.

Unfortunately, there was no straightforward way to separate NIOS V from the GDB environment. 

## How single-stepping works
The software-based stepping algorithm is as follows:
1. Read the instruction at the current PC
2. Determine its possible next address
3. For branches and jumps, calculate the appropriate destination
4. Temporarily place a breakpoint at the next instruction
5. Resume the debugee
6. When the temporary breakpoint is reached, remove it.
7. Return control to the command UI. 

## Trap Frame
The trap frame saves the following registers:
* General-purpose registers (`x1` - `x31`)
* `mepc`
* `mcause`
* `mstatus`
* `mtval`

## Startup and exeucution flow
```
_start
  │
  ▼
Set stack pointer
  │
  ▼
Initialize VGA and breakpoint state
  │
  ▼
Patch program_start
  │
  ▼
Run debugee
  │
  ▼
Breakpoint reached
  │
  ▼
Save TrapFrame
  │
  ▼
Debugger command loop
  │
  ▼
Resume or step
```

## Commands
| Command | Purpose | Example |
|---|---|---|
| `continue` | Resume until the next breakpoint | `continue` |
| `step [count]` | Execute one or more instructions | `step 5` |
| `break <address>` | Set a breakpoint | `break 0x100` |
| `delete <index>` | Remove a breakpoint | `delete 0` |
| `breakpoints` | List active breakpoints | `breakpoints` |
| `regs [register]` | Display registers | `regs a0` |
| `list` | Disassemble instructions | `list` |
| `restart` | Restart the debugee | `restart` |
| `help` | Show available commands | `help` |

## Demo 

https://github.com/user-attachments/assets/54933605-7041-43d0-be1e-69fc40bad433

## Source code
The source code for this project is not publicly available due to University of Toronto course regulations. This repository contains project documentation and a demonstration of the completed system.


