# ABI Specifications

This repository contains specifications for multi-architecture backend infrastructure, including paging, calling conventions, memory protection, and other ABI-related details.

## Supported Architectures

- **RISC-V**: Specifications in [riscv/](riscv/) - See [riscv/PAGING.md](riscv/PAGING.md) and [riscv/TESTING.md](riscv/TESTING.md)
- **Additional architectures**: To be added

## Emulator-Specific Supervisor Mode

The specifications defined here, particularly for paging and security features like Control Flow Integrity (CFI), are designed for user-space emulation environments. They are architecture-specific but share common design principles, and may not be compliant with standard privileged specifications for their respective architectures.

The goal is to create lightweight, "alternate supervisor mode" implementations tailored for emulators and JIT compilers. This approach allows for advanced memory protection and security features to be implemented at the user-space level, without requiring full-blown privileged hardware support.

Key characteristics of this emulator-native supervisor mode include:
- **User-Space Implementation**: All features are designed to be implemented within a user-level process.
- **Flexibility**: The specifications are adaptable to different host environments, such as WebAssembly or native x86-64.
- **Performance**: The designs prioritize performance in an emulation context.
- **Security**: The non-standard CFI and permission models provide a security framework suited for running untrusted guest code in a sandboxed environment.
- **Multi-Architecture**: Each guest architecture can have its own specifications while sharing common design patterns.

This non-standard approach is a deliberate design choice to meet the unique requirements of high-performance, secure, user-space emulation across multiple guest architectures.

## Goals
- [ ] Add project goals

## Progress
- [ ] Initial setup

---
*AI assisted*
