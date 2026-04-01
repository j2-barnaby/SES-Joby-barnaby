# ARM Cortex M3 | STM32F100 | Embedded Systems  
  
## Worksheet 2 — Build, Flash, and Debug Workflow (Make, OpenOCD & GDB)
  
**Student Name:** Joby Barnaby 
**Student ID:** 24049911

---
## Overview

This worksheet covers the full workflow for building, loading, and debugging a C program on the STM32F100 microcontroller. You will use `make` to compile, OpenOCD to interface with the board, and GDB to step through code and inspect memory.

**Files you'll work with:**

| File                  | Purpose                                                  |
| --------------------- | -------------------------------------------------------- |
| `main.c`              | Your C source code — sets up timer and interrupt handler |
| `Makefile`            | Build instructions                                       |
| `demo.elf`            | Compiled executable (created by `make`)                  |
| `openocd.cfg`         | OpenOCD configuration                                    |
| `startup_stm32f10x.c` | System startup code (do not modify)                      |
| `stm32f100.ld`        | Linker script — defines memory layout                    |
| `stm32f10x_conf.h`    | Configuration header                                     |

---
## Repository Setup

```bash

cd ~/SES

git clone https://gitlab.uwe.ac.uk/c-duffy/ses-worksheet-2.git

cd ses-worksheet-2/worksheet2

```

**Verify your location:**

```bash

pwd

# Should show: /home/your-username/SES/ses-worksheet-2/worksheet2

```

---
## The Source Code

**`main.c` contains a simple program that increments a variable forever:**

```c

int i = 0;

int off = 5;

  

void inc(void)

{

    i += off;

}

  

int main(void)

{

    while (1) {

        inc();

    }

}

```


**What this does:** 
`i` starts at 0 and increases by 5 on every call to `inc()` — so it counts 0, 5, 10, 15, 20... indefinitely. This gives us something predictable to observe in the debugger.

**Where things live in memory after compilation:**

- `main()` and `inc()` functions → Flash at `0x08000xxx`

- Variables `i` and `off` → RAM at `0x20000xxx`

---
## Building the Program

```bash

make

```


**This runs four steps:**

1. **Compile `main.c`** → `main.o`

2. **Compile `startup_stm32f10x.c`** → `startup_stm32f10x.o`

3. **Compile `system_stm32f10x.c`** (from the ST library) → `system_stm32f10x.o`

4. **Link all `.o` files** → `demo.elf`

**Verify the build succeeded:**

```bash

ls -l demo.elf

# Should show a file around 76KB

```

**To clean and rebuild from scratch:**

```bash

make clean

make

```

![[Pasted image 20260330165510.png]]

---
## Loading the Program onto the Board

Three terminals are used simultaneously. Open them all before starting.

### Terminal 1 — Start OpenOCD

```bash

cd ~/SES/ses-worksheet-2/worksheet2

openocd -f openocd.cfg

```

**Leave this running. You should see:**

```

Info : Listening on port 4444 for telnet connections

Info : Listening on port 3333 for gdb connections

```

![[Pasted image 20260330165731.png|697]]

### Terminal 2 — Flash via Telnet

```bash

cd ~/SES/ses-worksheet-2/worksheet2

telnet localhost 4444

```

At the `>` prompt:

```

reset halt

stm32f1x mass_erase 0

flash write_bank 0 demo.elf 0

```

**Expected output after the flash command:**

```

wrote 76521 bytes from file demo.elf to flash bank 0 at offset 0x00000000

```

The program is now permanently stored in the board's flash memory at `0x08000000`.

![[Screenshot 2026-03-30 165923.png]]

---
## Debugging with GDB

### Terminal 3 — Start GDB

```bash

cd ~/SES/ses-worksheet-2/worksheet2

gdb-multiarch demo.elf

```

You should see the `(gdb)` prompt.

### Connect to the Board

```

target extended-remote :3333

```

This connects GDB to OpenOCD (which is bridged to the board on port 3333).
### Load the Program

```

load

```

![[Pasted image 20260330172409.png|697]]

---
## Setting Breakpoints and Running

**Set breakpoints at both functions:**

```

break main

break inc

```

**Start execution:**

```

continue

```

**The program will run and stop at `main()`:**

```

Breakpoint 1, main () at main.c:8

8       while (1) {

```

![[Pasted image 20260330172305.png|697]]

---
## Inspecting Variables

**Print the current value of `i`:**

```

print i

```

```

$1 = 0

```

**Print `off`:**

```

print off

```

```

$2 = 5

```

**Continue to the next breakpoint (inside `inc()`):**

```

continue

```

```

Breakpoint 2, inc () at main.c:4

4           i += off;

```

**The program is now paused at line 4, *before* executing `i += off`. Step over it:**

```

step

```

**Print `i` again to confirm it changed:**

```

print i

```

```

$3 = 5

```

Continue once more and check again — you'll see `i` is now 10, confirming the loop is working.

![[Pasted image 20260330172223.png|697]]

---
## Examining CPU Registers

**View the Stack Pointer:**

```

print /x $sp

```

```

$5 = 0x20004fe0

```

**View all registers at once:**

```

info registers

```

**Key registers to understand:**

| Register | Example Value | Meaning         |     |
| -------- | ------------- | --------------- | --- |
| `pc`     | `0x08000190`  | Program Counter |     |
| `sp`     | `0x20004fe0`  | Stack Pointer   |     |
| `lr`     | `0x0800019f`  | Link Register   |     |
![[Pasted image 20260330172958.png|697]]


**Overall screenshot:** 
![[Pasted image 20260330173151.png|697]]

---
## Automated Breakpoint Actions

You can make GDB print a variable and continue automatically every time a breakpoint is hit, which is useful for tracing a value over many iterations:

```

break inc

commands

silent

print i

continue

end

```

Then:

```

continue

```

You'll see `i` printing continuously:

```

$6 = 10

$7 = 15

$8 = 20

$9 = 25

...

```

Press `Ctrl+C` to stop.

![[Pasted image 20260330175051.png]]

---
## Memory Map

### Flash (program storage) — `0x08000000` to `0x0803FFFF`

| Address      | Contents                    |
| ------------ | --------------------------- |
| `0x08000000` | Stack pointer initial value |
| `0x08000004` | Reset vector                |
| `0x08000188` | Startup code entry          |
| `0x08000190` | TIM2_IRQHandler             |
| `0x0800019c` | main()                      |
  
### RAM (variable storage) — `0x20000000` to `0x2000FFFF`

|Address|Contents|
|---|---|
|`0x20000xxx`|`counter`|
|`0x20004fe0`|Stack pointer|

---

  

## File Structure

```

ses-worksheet-2/

├── STM32F10x_StdPeriph_Lib_V3.5.0/

│   └── Libraries/

│       ├── CMSIS/CM3/DeviceSupport/ST/STM32F10x/

│       │   └── system_stm32f10x.c      ← Used during build

│       └── STM32F10x_StdPeriph_Driver/

└── worksheet2/                         ← YOUR WORKING DIRECTORY

    ├── main.c                          ← Source code (edit this)

    ├── Makefile                        ← Build instructions

    ├── openocd.cfg                     ← OpenOCD config

    ├── startup_stm32f10x.c             ← Startup code (do not edit)

    ├── stm32f100.ld                    ← Linker script

    ├── stm32f10x_conf.h                ← Config header

    ├── main.o                          ← Created by make

    ├── startup_stm32f10x.o             ← Created by make

    ├── system_stm32f10x.o              ← Created by make

    └── demo.elf                        ← Final executable

```

---

## Workflow Summary

Every time you edit and re-test code:

```bash

# 1. Edit your code

nano main.c

  

# 2. Build

make clean && make

  

# 3. Flash (Terminal 2 - Telnet)

reset halt

stm32f1x mass_erase 0

flash write_bank 0 demo.elf 0

  

# 4. Debug (Terminal 3 - GDB)

load

break main

continue

print i

```

---

## GDB Quick Reference

|Command|Purpose|
|---|---|
|`target extended-remote :3333`|Connect to board via OpenOCD|
|`load`|Load `demo.elf` into flash|
|`break main`|Set breakpoint at `main()`|
|`break TIM2_IRQHandler`|Set breakpoint at interrupt handler|
|`continue`|Run until next breakpoint|
|`step`|Execute one line|
|`print counter`|Print value of counter variable|
|`print /x $sp`|Print stack pointer in hex|
|`info registers`|Show CPU registers|
|`Ctrl+C`|Pause execution|

---
## Key Concepts

**Build pipeline:**

```

main.c → (compiler) → main.o → (linker) → demo.elf → (flash) → Running on board

                                                            ↑

                                                      GDB inspects here

```

**Why three terminals?** OpenOCD must stay running as a bridge between your PC and the board. Telnet sends flash commands to OpenOCD. GDB connects separately for debugging — they all run simultaneously.

**Flash vs RAM:** Flash stores the program permanently (survives power off). RAM stores variables while the program runs (lost on reset). The linker script `stm32f100.ld` defines where each section goes.

**Breakpoints** pause the CPU at a specific address in flash. When paused, you can read RAM variables, CPU registers, and step through instructions one at a time.

 ---
## Conclusion

This worksheet demonstrated the complete workflow for developing, flashing, and debugging a simple C program on the STM32F100 microcontroller using OpenOCD and GDB. By building `demo.elf` with `make`, loading it into the board’s flash, and using breakpoints in GDB, we were able to:

- Observe variable changes (`i`) in real time.
- Trace program execution step by step.
- Examine CPU registers and memory layout.
- Automate variable printing at breakpoints to monitor iterative updates.

Through this process, we gained practical experience with **embedded development workflows**, **debugging techniques**, and **memory management on microcontrollers**, reinforcing the understanding of how code translates into hardware behavior.