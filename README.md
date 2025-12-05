# vmtrace-triton-tools

Small Triton-based helpers for analyzing VMProtect-style virtual machine handlers from execution traces.

The scripts are designed around the instruction trace format used in Jonathan Salwan’s
[`VMProtect-devirtualization`](https://github.com/JonathanSalwan/VMProtect-devirtualization) project:
they consume the same `r:` / `mr:` / `i:` trace structure (produced via Intel PIN or similar DBI tools),
then symbolically replay a handler window and summarize how “VM register slots” (`vm_0`, `vm_1`, …)
flow into real registers and stack locations.

Instead of manually stepping through hundreds of instructions in a debugger, you can point the script
at a handler PC and immediately see which VM slots are used, how they are combined, and where they end up.

---

## Features

- Salwan-style trace parsing  
  Supports traces that contain:
  - `r:` register snapshots
  - `mr:` memory write events
  - `i:` instruction bytes

- Automatic handler window extraction  
  - Finds the first instruction at the given handler PC  
  - Uses the previous `r:` event as the initial CPU snapshot  
  - Replays up to `N` instructions starting from that point

- VM slot modeling around EBP  
  - Defines a “VM register file” region around `ebp`  
    (default: `[ebp - 0x40, ebp + 0x40)`)  
  - Each 4-byte word in that region is a separate symbolic variable:
    `vm_0`, `vm_1`, `vm_2`, …

- Symbolic execution with Triton  
  - Replays the handler window through Triton  
  - Tracks how operations transform the symbolic `vm_i` slots  
  - Preserves symbolic expressions wherever possible

- Human-readable summaries  
  At the end of the run, the script prints:

  1. Register summary  
     For each of `eax`, `ebx`, `ecx`, `edx`, `esi`, `edi`, `ebp`, `esp`:
     - If purely concrete: final 32-bit value  
     - If symbolic: raw AST and a simplified expression

  2. VM slot summary  
     For each 4-byte slot in the VM region (for example `[ebp-0x40]`, `[ebp-0x3c]`, …):
     - If any byte is concrete: the 32-bit concrete value  
     - If fully symbolic: the raw concatenation and a simplified form  
       (often just `vm_n` or a simple expression of `vm_*`)

This is aimed at quickly answering questions such as:

- Which VM slot is the current VM PC?
- Which handler reads from `vm_0` and writes back into `vm_0` with an offset?
- Which slots are used as immediates for jumps or arithmetic?

---

## Typical workflow

1. Collect a trace

   Use an Intel PIN tool (or any DBI) to record:
   - `r:` register snapshots  
   - `mr:` memory writes  
   - `i:` instruction bytes  

   Save the result as something like:

   ```text
   vmtrace.out

2. Identify a handler entry

   From static analysis or debugger work, pick a VM handler address, for example:

   ```text
   0x0043641a
   ```

3. Run the script

   ```bash
   python3 vm_handler_symbolic.py vmtrace.out 0x0043641a --max-insn 80
   ```

4. Inspect the output

   * `=== register summary ===` shows which registers depend on `vm_i`
   * `=== vm slot summary ===` shows which stack slots correspond to which `vm_i`,
     and whether they stayed symbolic or collapsed to constants

From there, you can start mapping handlers to higher-level semantics, such as:

* `vm_0` is the VM program counter
* `vm_1` is the data stack top
* This handler implements patterns like `vm_1 = vm_1 + vm_2`

---

## Why this helps reverse engineering

Manual devirtualization usually means:

* Stepping through traces in a debugger
* Tracking stack slots and temporary values by hand
* Reconstructing VM register flows on paper

This project automates most of that bookkeeping:

* Restores the exact handler entry state from the trace
* Models the VM stack region as symbolic variables
* Lets Triton propagate those variables through the handler
* Produces compact expressions that show how the handler transforms `vm_i`

The goal is to provide a faster, more systematic way to:

* Decide which handlers are worth deeper analysis
* Identify inputs and outputs of each handler
* Build a higher-level devirtualizer or IR recompiler on top of these summaries

---

## Requirements

* Python 3
* Triton (Python bindings installed)
* A trace file in the `r:` / `mr:` / `i:` format used by
  [`VMProtect-devirtualization`](https://github.com/JonathanSalwan/VMProtect-devirtualization)

---

## References and inspiration

Core ideas and workflow were inspired by🙏:

* Jonathan Salwan, VMProtect-devirtualization
  [https://github.com/JonathanSalwan/VMProtect-devirtualization](https://github.com/JonathanSalwan/VMProtect-devirtualization)  

* Secret Club, “VMProtect: LLVM Lifting” series
 [https://secret.club/2021/09/08/vmprotect-llvm-lifting-1.html](https://secret.club/2021/09/08/vmprotect-llvm-lifting-1.html)  
 [https://secret.club/2021/09/08/vmprotect-llvm-lifting-2.html](https://secret.club/2021/09/08/vmprotect-llvm-lifting-2.html)  
 [https://secret.club/2021/09/08/vmprotect-llvm-lifting-3.html](https://secret.club/2021/09/08/vmprotect-llvm-lifting-3.html)  

* (Background reading, Korean)
  “VMProtect의 역공학 방해 기능 분석 및 Pin을 이용한 우회 방안”
  정보처리학회논문지: 컴퓨터 및 통신시스템(KTCCS), 2021, vol.10, no.11, pp.297–304

```
::contentReference[oaicite:0]{index=0}
```
