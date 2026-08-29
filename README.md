# ARM Cortex‑M Programming — Bare‑Metal Architecture Labs

**Register‑level, HAL‑free exploration of the ARM Cortex‑M4 core and its exception model, built and debugged on real STM32F411 silicon.**

[![MCU](https://img.shields.io/badge/MCU-STM32F411CEUx-03234B?logo=stmicroelectronics&logoColor=white)](https://www.st.com/en/microcontrollers-microprocessors/stm32f411ce.html)
[![Core](https://img.shields.io/badge/Core-ARM%20Cortex--M4-0091BD?logo=arm&logoColor=white)](https://developer.arm.com/Processors/Cortex-M4)
[![Language](https://img.shields.io/badge/Language-C%20%2F%20ARM%20Assembly-A8B9CC?logo=c&logoColor=white)]()
[![Toolchain](https://img.shields.io/badge/Toolchain-STM32CubeIDE%20%2F%20GNU%20ARM%20GCC-informational)]()
[![HAL](https://img.shields.io/badge/Abstraction-None%20(Bare--Metal)-critical)]()

---

## Table of Contents

- [About This Repository](#about-this-repository)
- [Why Bare‑Metal, Why This Matters](#why-baremetal-why-this-matters)
- [Skills Demonstrated](#skills-demonstrated)
- [Target Hardware & Toolchain](#target-hardware--toolchain)
- [Repository Structure](#repository-structure)
- [Module‑by‑Module Breakdown](#module-by-module-breakdown)
- [Core Concepts Map](#core-concepts-map)
- [Building & Running](#building--running)
- [Viewing Program Output (Semihosting)](#viewing-program-output-semihosting)
- [Sample Output](#sample-output)
- [Roadmap](#roadmap)
- [Author](#author)
- [License](#license)

---

## About This Repository

This repository is a structured, incremental study of the **ARM Cortex‑M4 processor architecture**, implemented entirely at the **register level** — no HAL, no CMSIS drivers, no vendor abstraction layers. Every peripheral and core register (NVIC, SCB, CONTROL, PSP/MSP, SHCSR, UFSR, CCR, IPR…) is accessed directly through its memory‑mapped address, exactly as described in the *ARM Cortex‑M4 Technical Reference Manual* and the *ARMv7‑M Architecture Reference Manual*.

Each numbered folder is a **self‑contained STM32CubeIDE project** that isolates one architectural concept, deliberately triggers the processor behavior under study (an interrupt, a fault, a privilege violation, a stack‑pointer switch), and prints diagnostic output back to the host over semihosting so the behavior can be observed and verified on real hardware — not just simulated.

The goal is not to build an application. It is to build a working, hands‑on understanding of **how a Cortex‑M core actually behaves underneath the frameworks and RTOSes** that most embedded engineers use every day — the same knowledge required to debug hard faults in production firmware, write a syscall layer, port an RTOS, or reason about interrupt latency.

## Why Bare‑Metal, Why This Matters

Modern embedded development leans heavily on HAL libraries and RTOS abstractions, which is efficient — but it also hides the mechanics that matter most when things go wrong: a mis-stacked exception frame, an unexpected privilege escalation, a priority inversion, an unaligned access that trips a UsageFault. This repository intentionally strips all of that away and works directly against the silicon, so that each experiment answers a concrete question such as:

- *What actually happens to the stack and the Program Counter when an exception fires?*
- *How does the CPU distinguish Thread mode from Handler mode, and privileged from unprivileged code?*
- *How is an `SVC` instruction turned into a full system‑call dispatcher, the same mechanism every RTOS kernel (FreeRTOS, Zephyr, embOS…) uses under the hood?*
- *How does the NVIC decide which of two simultaneously pending interrupts runs first?*

## Skills Demonstrated

| Category | Specific Skills |
|---|---|
| **Processor Architecture** | Thread vs. Handler mode, Privileged vs. Unprivileged execution, `CONTROL` register, EPSR `T`-bit / Thumb state enforcement |
| **Memory System** | Bit‑banding (atomic bit‑level read/modify/write via the alias region), memory‑mapped peripheral access |
| **Stack Management** | Main Stack Pointer (MSP) vs. Process Stack Pointer (PSP), manual stack‑pointer relocation, exception stack‑frame layout (R0‑R3, R12, LR, PC, xPSR) |
| **Interrupt Handling (NVIC)** | Enabling/disabling IRQs (`ISER`/`ICER`), manually pending interrupts (`ISPR`), interrupt priority configuration (`IPR`), priority‑based preemption between peripherals |
| **Exception Model** | Full exception entry/exit sequence, `SVC` (Supervisor Call) exceptions, building an **SVC‑based system‑call dispatcher** with an operation table |
| **Fault Diagnostics** | Enabling and handling `HardFault`, `MemManage`, `BusFault`, and `UsageFault` via `SHCSR`; decoding `UFSR`; forcing and diagnosing an undefined‑instruction fault and a divide‑by‑zero trap (`DIV_0_TRP` in `CCR`) |
| **Low‑Level C & Assembly** | GNU inline assembly (`__asm volatile`, constraints, `naked` functions), reading/writing special registers (`MRS`/`MSR`), mixed C/assembly linkage |
| **Toolchain Literacy** | Linker scripts (`.ld`), startup files and the vector table, `arm-none-eabi-gcc`, STM32CubeIDE project configuration, semihosting-based host I/O |

## Target Hardware & Toolchain

| Item | Detail |
|---|---|
| **MCU** | STM32F411CEUx — Arm® Cortex‑M4 (no FPU used; built with soft float ABI) |
| **Typical board** | Any STM32F411CEUx‑based board (e.g. WeAct "Black Pill" STM32F411CEU6) |
| **IDE / Build system** | STM32CubeIDE (Eclipse CDT + `arm-none-eabi-gcc`) |
| **Debug probe** | ST‑Link/V2 (or compatible SWD probe) |
| **Debug output** | ARM Semihosting (`initialise_monitor_handles()` + `printf`) |
| **Abstraction layer** | **None** — direct register access only, no HAL/LL/CMSIS drivers |
| **Language** | C (register access, control flow) + ARM Thumb‑2 assembly (inline and standalone, GNU syntax) |

## Repository Structure

```
ARM-Cortex-M-Programming/
├── 003_operation_modes/          # Thread mode ↔ Handler mode via a software interrupt
├── 004_inline_1/                 # GNU inline-assembly primitives (LDR/STR, MOV, MRS/MSR)
├── 005_access_levels/            # Privileged ↔ Unprivileged execution via CONTROL[0]
├── 006_T-bit/                    # EPSR T-bit / Thumb-state enforcement, invalid-state fault
├── 007_bit_banding/              # Atomic single-bit access via the bit-band alias region
├── 008_stack_pointer/            # MSP → PSP relocation with a naked function
├── 009_USART2_initial_pending/   # Manually pending an NVIC interrupt bit (ISPR)
├── 010_interrupt_priority/       # NVIC IPR configuration & priority-based preemption
├── 011_exception_entry_exit/     # PSP-mode exception entry/exit behavior
├── 012_fault_exception/          # HardFault / MemManage / BusFault / UsageFault handling
├── 013_SVC_number/               # Extracting the SVC immediate from the exception frame
├── 014_SVC_operation/            # SVC-based syscall dispatcher (add/sub/mul/div)
└── .gitignore
```

Each folder is an independent STM32CubeIDE project and contains its own:
- `Src/main.c` — the experiment
- `Src/syscalls.c`, `Src/sysmem.c` — minimal libc/semihosting glue (auto‑generated)
- `Startup/startup_stm32f411ceux.s` — reset handler, vector table, `.data`/`.bss` init
- `STM32F411CEUX_FLASH.ld` / `STM32F411CEUX_RAM.ld` — linker scripts
- `.project` / `.cproject` — STM32CubeIDE project metadata
- `*Debug.launch` / `*Debug.cfg` — pre‑configured debug session (ST‑Link, semihosting enabled)

## Module‑by‑Module Breakdown

| # | Module | Concept | What It Does |
|---|---|---|---|
| 003 | `operation_modes` | Thread vs. Handler mode | Enables `IRQ3` in the NVIC and fires it via the Software Trigger Interrupt Register (`STIR`), proving via `printf` output that execution moves from Thread mode into Handler mode and back. |
| 004 | `inline_1` | GNU inline assembly | A set of isolated exercises (load/store, `MOV`, `MRS`) covering register constraints, operand syntax, and reading the `CONTROL` special register from C — the foundation used by every later module. |
| 005 | `access_levels` | Privilege levels | Reads, modifies, and writes bit 0 of `CONTROL` to drop the core from Privileged to **Unprivileged** Thread mode at runtime, then observes the effect on interrupt behavior and fault handling. |
| 006 | `T-bit` | Thumb state (EPSR) | Deliberately branches to an address with bit 0 cleared (ARM state marker) on a Thumb‑only core, forcing an **invalid‑state UsageFault**, and captures it in `HardFault_Handler`. |
| 007 | `bit_banding` | Bit‑band alias region | Compares a normal read‑modify‑write bit clear against a single **atomic store** through the bit‑band alias address, computed from the official `bit_band_base + (byte_offset × 32) + (bit_number × 4)` formula. |
| 008 | `stack_pointer` | MSP / PSP | A `naked` function computes a new stack top in SRAM, switches the active stack pointer from MSP to **PSP** via `CONTROL[1]`, runs a normal function call on the new stack, then triggers an `SVC` exception to show the automatic **PSP → MSP** switch on exception entry. |
| 009 | `USART2_initial_pending` | NVIC pending bit | Manually sets the USART2 IRQ's pending bit in `ISPR1` *before* enabling it in `ISER1`, demonstrating that an interrupt fires as soon as it becomes enabled if already pending — independent of the peripheral itself. |
| 010 | `interrupt_priority` | NVIC priority (`IPR`) | Assigns different priority levels to `TIM2` and `I2C1_EV`, then pends `TIM2` first; from inside `TIM2`'s handler it pends the higher‑priority `I2C1_EV` interrupt and shows it **preempting** the lower‑priority handler mid‑execution. |
| 011 | `exception_entry_exit` | Exception stacking | Switches to PSP mode before generating a software interrupt, exposing exactly how the processor automatically stacks context on the **active** stack pointer during exception entry/exit. |
| 012 | `fault_exception` | Fault diagnostics | Enables `MemManage`, `BusFault`, and `UsageFault` in `SHCSR`, then (in two variants) forces an **undefined‑instruction fault** and a **divide‑by‑zero trap** (`DIV_0_TRP` bit in `CCR`). A `naked` `UsageFault_Handler` grabs the stack frame and dumps `UFSR` plus every stacked register (`R0–R3`, `R12`, `LR`, `PC`, `xPSR`) for post‑mortem analysis. |
| 013 | `SVC_number` | SVC mechanism | Executes `SVC #5`, then in a `naked` handler retrieves the stacked return address, walks back to the `SVC` opcode in Flash to **extract the immediate operand**, modifies it, and writes the result back into the stacked `R0` so the caller receives it on return. |
| 014 | `SVC_operation` | Syscall dispatcher | Builds on Module 013 to implement a genuine **system‑call table**: four C wrapper functions (`add`, `sub`, `mul`, `div`) each issue a distinct `SVC` number, and a single `SVC_Handler_c` decodes the number, performs the operation on the stacked arguments, and returns the result — the same pattern used by RTOS kernels to implement `taskYIELD()`‑style privileged calls from user code. |

## Core Concepts Map

A quick cross‑reference to the ARM Cortex‑M4 architecture topics this repository covers hands‑on:

- **Operating modes:** Thread mode / Handler mode
- **Privilege levels:** Privileged / Unprivileged, via `CONTROL[0]`
- **Stack model:** MSP, PSP, `CONTROL[1]`, exception stack frame
- **Instruction state:** Thumb `T`-bit (EPSR), invalid‑state fault
- **Memory system:** Bit‑band region and alias address computation
- **NVIC:** `ISER`/`ICER` (enable), `ISPR`/`ICPR` (pending), `IPR` (priority), preemption
- **Exception model:** Entry/exit sequence, tail‑chaining conditions, vector table
- **Fault handling:** `SHCSR`, `UFSR`, `CFSR`, `CCR` (`DIV_0_TRP`), `HardFault`/`MemManage`/`BusFault`/`UsageFault`
- **System calls:** `SVC` instruction, immediate‑operand extraction, syscall dispatch tables

## Building & Running

### Prerequisites
- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) (recommended — projects are pre‑configured for it), **or** a standalone `arm-none-eabi-gcc` toolchain
- An ST‑Link/V2 (or compatible) SWD debug probe
- A board built around the STM32F411CEUx

### Option A — STM32CubeIDE (recommended)
1. Clone the repository:
   ```bash
   git clone https://github.com/ShayanSaed/ARM-Cortex-M-Programming.git
   ```
2. In STM32CubeIDE: **File → Import → Existing Projects into Workspace**, and point it at the repository root (each numbered folder will be discovered as a separate project).
3. Select the module you want to run, build it (`Ctrl+B`), and start a debug session using the project's pre‑configured `*.launch` file — this automatically enables semihosting on the debug console.

### Option B — Command line (GNU ARM toolchain)
```bash
cd 012_fault_exception
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfloat-abi=soft \
    -O0 -g3 -specs=nosys.specs -specs=rdimon.specs \
    -T STM32F411CEUX_FLASH.ld \
    Startup/startup_stm32f411ceux.s Src/main.c Src/syscalls.c Src/sysmem.c \
    -o firmware.elf

openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
    -c "program firmware.elf verify reset exit"
```

## Viewing Program Output (Semihosting)

Every module that calls `initialise_monitor_handles()` routes `printf` through **ARM semihosting** rather than a UART, so output appears directly in the debugger console:

- **STM32CubeIDE:** the pre‑supplied `*.launch`/`*.cfg` files already enable semihosting — output appears in the *Console* view during a debug session.
- **OpenOCD / GDB manually:** connect with `arm-none-eabi-gdb`, then run:
  ```gdb
  monitor arm semihosting enable
  ```
  before continuing execution.

## Sample Output

Running **`010_interrupt_priority`** on target produces:

```
--- TIM2_IRQHandler ---
--- I2C1_EV_IRQHandler ---
```

Running **`012_fault_exception`** (forced UsageFault variant) produces a full register dump for post‑mortem debugging:

```
UsageFault exception!
UFSR: 1
pBaseStackFrame: 0x20017fc0
Value of R0: 0
Value of R1: 0
...
Value of PC: 20010000
Value of xPSR: 61000000
```

## Roadmap

Planned additions as the study of the Cortex‑M architecture continues:
- [ ] PendSV‑based context switch (minimal cooperative scheduler)
- [ ] SysTick‑driven time base and tick‑hook

## Author

**Shayan Saed**
GitHub: [@ShayanSaed](https://github.com/ShayanSaed)

This repository was built as a hands‑on, from‑the‑register‑map‑up study of the ARM Cortex‑M4 architecture, in preparation for embedded software engineering roles that require debugging and reasoning about firmware below the HAL/RTOS layer.

## License

This project is licensed under the [MIT License](LICENSE) — feel free to use these exercises as a learning reference.
