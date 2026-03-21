# r5-abi-specs

Specifications for the virtual memory and ABI layer used by a family of RISC-V backend implementations (rift, r52x, speet, vane). These backends compile or emulate RISC-V guest code targeting WebAssembly, x86-64, and JavaScript runtimes.

This is a documentation-only repository. It contains no runnable code.

## What this repo is

The backends (rift, r52x, speet, vane) share a common virtual memory model that differs deliberately from the standard RISC-V privileged specification. This repo documents that model so it can be implemented consistently across all backends. The specs are non-normative with respect to upstream RISC-V — they define a custom supervisor-mode-like layer that runs entirely in user space.

## Repository layout

```
riscv/
  PAGING.md     -- Core paging system specification (primary document)
  TESTING.md    -- Guide for writing and running tests against implementations
  README.md     -- RISC-V section overview

r5-abi-specs/
  harness/
    README.md   -- Placeholder; no harness code exists yet

README.md       -- This file
```

## Paging system (riscv/PAGING.md)

The central document. It specifies the unified virtual memory model shared by rift, r52x, speet, and vane.

Key design decisions:

- **Page size: 64 KiB** (65536 bytes). All pages and page tables are aligned to 64KB boundaries.
- **64-bit virtual addresses**, split into 16-bit fields at bits [63:48], [47:32], [31:16], and [15:0].
- **Two translation modes**: single-level (flat array) and multi-level (3-level hierarchy). The flat mode maps page numbers directly; the hierarchical mode walks three 2^16-entry tables.
- **Two physical address sizes**: 32-bit (4 GiB space, 4-byte PTEs) and 64-bit (16 EiB space, 8-byte PTEs).
- **Security directory**: A separate table, indexed from the low bits of each page table entry, that stores per-page R/W/X permission flags, a CFI bitmask, and the top bits of the physical page base address. Every address translation requires one additional load to read the security entry.
- **CFI metadata**: The security entry includes a bitmask used for control flow integrity checks on executable pages.

The security directory is the main structural difference from a conventional page table. The physical base address of a page is split: the lower bits live in the PTE, and the top bits (8 for 32-bit mode, 16 for 64-bit mode) live in the security entry. This allows permissions and CFI data to be associated with physical pages without enlarging the PTE.

## Status

- The paging specification is written and has gone through several revision cycles (see git log).
- No test harness code exists yet; `r5-abi-specs/harness/` is a placeholder.
- `riscv/TESTING.md` documents the intended testing approach (Vitest for JS, `cargo test` for Rust) but lists no concrete test cases — this section is explicitly marked as a TODO.
- Only RISC-V is specified. The top-level README previously mentioned "additional architectures to be added" but no other architecture directories exist.

## Implementations covered by these specs

The PAGING.md document references four backends that are expected to implement this spec. They live in separate repositories:

| Backend | Target | Notes |
|---------|--------|-------|
| rift    | WebAssembly | Mapper accepts `security_directory_base` |
| r52x    | x86-64 | Uses `MapperCallback` type |
| speet   | WebAssembly (recompiler) | `set_mapper_callback()` on `RiscVRecompiler` |
| vane    | JavaScript/WASM JIT | `Mem` struct holds `shared_security_directory_vaddr` |

## Notes

- All specs are designed for user-space implementation. There is no privileged hardware dependency.
- The design is not compliant with the RISC-V privileged ISA specification and does not attempt to be.
