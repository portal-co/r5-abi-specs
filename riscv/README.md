# RISC-V ABI Specifications

This directory contains specifications for the RISC-V backend infrastructure.

## Documents

- **[PAGING.md](PAGING.md)**: Unified paging system specification used across all RISC-V backend implementations (rift, r52x, speet, vane). Defines the 64KB page-based virtual memory system with support for both single-level and multi-level page tables.

- **[TESTING.md](TESTING.md)**: Testing and harness guide for RISC-V implementations. Explains how to write and run tests, and how to integrate runtime/harness tooling.

## Overview

The RISC-V specifications are designed for a user-space RISC-V emulation environment and are not compliant with standard RISC-V privileged specifications. They provide a lightweight "alternate supervisor mode" tailored for emulators and JIT compilers, enabling advanced memory protection and security features at the user-space level.

## Key Features

- **64KB Page Size**: Large pages for efficient address translation
- **Multi-Level Paging**: Support for both flat and hierarchical page tables
- **Fine-Grained Permissions**: Read, Write, Execute permissions with CFI metadata
- **Flexible Physical Addressing**: Support for both 32-bit and 64-bit physical addresses
- **Cross-Platform**: Designed to work on WebAssembly, x86-64, and JavaScript backends

---
*AI assisted*
