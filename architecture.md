# Chez Scheme Architecture

Chez Scheme is a high-performance, bootstrapped Scheme compiler and runtime. This document describes the high-level architecture of the implementation.

## Repository Layout

```
ChezScheme/
├── s/           # Scheme compiler and standard libraries (~136k lines, 94 files)
├── c/           # C runtime kernel (~26k lines, ~49 files)
├── boot/pb/     # Pre-built portable bytecode boot files for bootstrapping
├── mats/        # Test suites (94 .ms test files)
├── makefiles/   # Build system templates and Zuo modules
├── csug/        # Chez Scheme User's Guide source
├── nanopass/    # Nanopass compiler framework (submodule)
├── zuo/         # Zuo build tool (submodule)
├── lz4/         # LZ4 compression (submodule)
├── zlib/        # zlib compression (submodule)
├── unicode/     # Unicode tables
├── stex/        # Documentation build tool
└── examples/    # Example programs
```

## Two-Layer Implementation

The implementation divides cleanly into two layers:

1. **Compiler + Libraries** (`s/`) — written in Scheme: the macro expander, all compiler passes, code generators, standard library, REPL editor, and boot file generation.
2. **Runtime Kernel** (`c/`) — written in C: garbage collector, memory allocator, OS interface, I/O, threading, FFI, fasl loader, and the entry point.

## Build System

The primary build tool is **Zuo** (`build.zuo`), a minimal task runner written in Scheme, wrapped by a traditional `Makefile` for convenience. Configuration is handled by the `configure` shell script, which generates a `Makefile` and `main.zuo` in a workarea directory named after the target machine type.

### Bootstrap

Chez Scheme is a **bootstrapped compiler**: you need Chez Scheme to build Chez Scheme. The repository ships pre-built **portable bytecode (pb)** boot files in `boot/pb/` to break the bootstrapping cycle on new machines.

Bootstrap flow:
1. C kernel is compiled with the host C compiler
2. The pb interpreter runs from pre-built `petite.boot` + `scheme.boot`
3. The full compiler rebuilds itself, producing native boot files for the target machine type
4. The kernel is relinked against native boot files

Two boot files are produced for every machine type:
- **`petite.boot`** — runtime libraries only (no compiler)
- **`scheme.boot`** — runtime + compiler (enables `compile-file`, `eval`, etc.)

## Compiler Pipeline

```
Scheme source (.ss)
        │
        ▼
  [syntax.ss]  ─── Macro expansion (psyntax / hygienic R6RS macros)
        │
        ▼
  [cp0.ss]     ─── Source optimizations: constant folding, dead code
                   elimination, beta reduction, inlining
        │
        ▼
  [cptypes.ss] ─── Type inference and type-directed optimizations
        │
        ▼
  [cpletrec.ss]─── letrec/letrec* transformation
        │
        ▼
  [cpcommonize.ss] Common subexpression elimination
        │
        ▼
  [cpnanopass.ss + cpprim.ss]
               ─── Main compiler: nanopass IR lowering,
                   register allocation, instruction selection,
                   primitive inlining
        │
        ▼
  [arch backend] ─ x86_64.ss / arm64.ss / x86.ss / arm32.ss / ppc32.ss / pb.ss
        │
        ▼
  [fasl.ss]    ─── Serialization to fasl (fast-load) format
        │
        ▼
  Compiled file (.so) or boot file (.boot)
```

The compiler uses the **Nanopass framework**, which represents transformations as a series of small passes over typed intermediate languages (`Lsrc`, `L1`, `L2`, … `Ln`). Each pass handles one concern, making individual passes easier to understand and test.

### Key Compiler Source Files

| File | Purpose |
|------|---------|
| `s/cmacros.ss` | Object layouts, type tags, global constants shared between compiler and C runtime |
| `s/syntax.ss` | Hygienic macro expander (psyntax) |
| `s/cpnanopass.ss` | Main compiler — IR lowering and code generation (~600KB) |
| `s/cpprim.ss` | Primitive inlining with type specialization (~430KB) |
| `s/cp0.ss` | Classic source optimizations (~310KB) |
| `s/cptypes.ss` | Type inference pass (~130KB) |
| `s/compile.ss` | `compile-file`, `compile-program`, public compilation API |
| `s/primdata.ss` | Primitive function declarations and signatures |
| `s/fasl.ss` | Fasl serialization; `s/vfasl.ss` for optimized variant |

### Architecture Backends

Each backend file defines machine-specific instruction selection and register allocation:

| File | Target |
|------|--------|
| `s/x86_64.ss` | x86-64 (all OSes) |
| `s/x86.ss` | x86-32 |
| `s/arm64.ss` | AArch64 |
| `s/arm32.ss` | ARMv6/ARMv7 |
| `s/ppc32.ss` | 32-bit PowerPC |
| `s/pb.ss` | Portable bytecode interpreter |

Machine-type `.def` files (e.g., `ta6le.def`, `tarm64nt.def`) supply platform-specific constants (alignment, available instructions, calling conventions) and select the backend. They are generated from `s/unix.def` / `s/tunix.def` templates.

## Runtime Kernel (`c/`)

### Key C Files

| File | Purpose |
|------|---------|
| `c/main.c` | Entry point, command-line parsing |
| `c/scheme.c` | Heap initialization, thread context (TC) setup |
| `c/gc.c` | Generational garbage collector |
| `c/alloc.c` | Object allocation |
| `c/segment.c` | Low-level memory segment management |
| `c/fasl.c` | Fasl deserialization and linking |
| `c/foreign.c` + `c/ffi.c` | Foreign function interface |
| `c/thread.c` | Threading (pthreads on Unix, Win32 threads on Windows) |
| `c/io.c` + `c/new-io.c` | I/O |
| `c/symbol.c` + `c/intern.c` | Symbol table and interning |

### Object Representation

Every Scheme value fits in a single machine word. The low-order bits encode the type tag:

- `#b001` — pair
- `#b010` — closure
- `#b011` — immediate values (fixnum, char, boolean, etc.)
- `#b111` — typed heap objects (vectors, strings, records, …)

Further discrimination of typed objects uses the first word of the heap object. This scheme enables type checks and field access with simple bit operations and avoids boxing overhead for common types.

### Garbage Collector

- **Strategy:** Generational copying GC with an optional mark-sweep phase for older generations.
- **Generations:** Up to 16 generations (configurable) plus a static (never-collected) generation.
- **Young objects** (gen 0) are copied to gen 1 on minor collection; older objects are eventually promoted or marked in place.
- **Special objects:** Weak pairs, ephemerons, and foreign segments are handled explicitly.
- **Parallel GC:** The `gc-par.inc` variant supports parallel marking.
- The GC implementation (`gc.c`, `gc-ocd.inc`, `gc-oce.inc`, `gc-par.inc`) is partly **generated** by `s/mkgc.ss`.

### Call Convention and Stack

Scheme code does **not** use the C stack. Instead:
- A heap-allocated **continuation stack** grows independently of the C stack.
- The **SFP** (Scheme Frame Pointer) register points to the current Scheme frame.
- The **AP** (Allocation Pointer) register is the bump pointer for heap allocation.
- These virtual registers live in the **Thread Context (TC)**, a per-thread C struct.
- Proper tail calls are implemented as direct jumps with no stack growth.

### Fasl Format and Linking

Compiled Scheme code is stored as **fasl** (fast-load) objects. Each code object carries relocation metadata. The fasl reader (`c/fasl.c`) resolves symbol references and patches relocations at load time — functioning as a lightweight linker. Boot files are fasl files with an additional header.

## Foreign Function Interface (FFI)

- `foreign-procedure` — call a C function from Scheme
- `foreign-callable` — make a Scheme procedure callable from C
- Supports automatic marshaling for primitive types (integers, floats, pointers, strings)
- Dynamic library loading via `dlopen`/`LoadLibrary`
- Hash-table based symbol lookup for both static and dynamically linked libraries

## Threading

- Full native thread support via pthreads (Unix) or Win32 threads (Windows), abstracted in `c/thread.h`.
- Each thread has its own **Thread Context (TC)** with independent allocation and GC state.
- Thread-safe mutexes and condition variables exposed to Scheme.
- The `t` prefix on machine type names (e.g., `ta6le`) indicates a threaded build.

## Platform Support

Machine types follow the naming convention `[t]<arch><os>`:

| Arch code | Architecture | OS code | OS |
|-----------|-------------|---------|-----|
| `a6` | x86-64 | `le` | Linux |
| `i3` | x86-32 | `osx` | macOS |
| `arm64` | AArch64 | `nt` | Windows |
| `arm32` | ARMv6/ARMv7 | `fb` | FreeBSD |
| `ppc32` | PowerPC32 | `ob` | OpenBSD |
| `rv64` | RISC-V 64 | `nb` | NetBSD |
| `la64` | LoongArch64 | `s2` | Solaris |
| `pb` | Portable bytecode | | |

The `t` prefix indicates threading enabled (e.g., `ta6le` = threaded x86-64 Linux).

## Test Infrastructure (`mats/`)

- 94 test suite files (`*.ms`), organized by language chapter and topic.
- Tests are compiled to `.mo` files and run under multiple configurations:
  - Optimization levels: `o=0` (safe) through `o=3` (unsafe)
  - With/without cp0 (`cp0=t`)
  - Interpreter mode (`eval=interpret`)
  - Primitive inlining suppressed (`spi=t`)
- C helper files (`foreign1.c`, `foreign2.c`) support FFI tests.
- Run with `make test` (supports `-j N` for parallel execution via Zuo).

## Key Reference Documents

- [`BUILDING`](BUILDING) — build instructions and configuration options
- [`IMPLEMENTATION.md`](IMPLEMENTATION.md) — deeper notes on compiler internals, linking, and modifying the system
- [Chez Scheme User's Guide](https://cisco.github.io/ChezScheme/csug/csug.html) — language and API reference
- [The Scheme Programming Language, 4th ed.](http://www.scheme.com/tspl4/) — R6RS language reference
