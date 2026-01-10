# R5 ABI Specifications

This repository contains specifications for the RISC-V backend infrastructure, including paging, calling conventions, and other ABI-related details.

## Emulator-Specific Supervisor Mode

The specifications defined here, particularly for paging and security features like Control Flow Integrity (CFI), are designed for a user-space RISC-V emulation environment. They are not compliant with standard RISC-V privileged specifications.

The goal is to create a lightweight, "alternate supervisor mode" tailored for emulators and JIT compilers. This approach allows for advanced memory protection and security features to be implemented at the user-space level, without requiring full-blown privileged hardware support.

Key characteristics of this emulator-native supervisor mode include:
- **User-Space Implementation**: All features are designed to be implemented within a user-level process.
- **Flexibility**: The specifications are adaptable to different host environments, such as WebAssembly or native x86-64.
- **Performance**: The designs prioritize performance in an emulation context.
- **Security**: The non-standard CFI and permission models provide a security framework suited for running untrusted RISC-V code in a sandboxed environment.

This non-standard approach is a deliberate design choice to meet the unique requirements of high-performance, secure, user-space RISC-V emulation.

## Goals
- [ ] Add project goals

## Progress
- [ ] Initial setup

---
*AI assisted*
