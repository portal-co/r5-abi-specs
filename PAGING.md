# RISC-V Backend Paging System Specification

This document specifies the unified paging system used across all RISC-V backend implementations (rift, r52x, speet, vane).

## Overview

All RISC-V backends share a common 64KB page-based virtual memory system with support for both single-level and multi-level page tables. This provides a consistent interface for memory address translation across different compilation targets (WebAssembly, x86-64, JavaScript).

A security directory, indexed from the page table entry, provides fine-grained permissions (Read, Write, Execute), Control Flow Integrity (CFI) metadata, and allows for arbitrary physical base addresses.

## Page Structure

### Common Specifications

- **Page Size**: 65536 bytes (64 KiB)
- **Page Alignment**: All pages aligned to 64KB boundaries
- **Virtual Address Format**: 64-bit virtual addresses
- **Physical Address Format**: Configurable - 32-bit or 64-bit physical addresses

### Address Layout

```
Virtual Address (64-bit)
├─ Level 3 Index: bits [63:48] - 16 bits (for multi-level)
├─ Level 2 Index: bits [47:32] - 16 bits (for multi-level)
├─ Level 1 Index: bits [31:16] - 16 bits (leaf level / page number)
└─ Page Offset:    bits [15:0]  - 16 bits (offset within page)
```

For single-level (flat) paging:
- Page Number: bits [63:16] - identifies the page
- Page Offset: bits [15:0] - offset within the page

## Physical Address Modes

All translation modes support two physical address sizes:

### 32-bit Physical Addressing
- **Address Space**: 4 GiB (2^32 bytes)
- **PTE Size**: 4 bytes (u32)
- **PTE Address**: `page_table_base + (index * 4)`

### 64-bit Physical Addressing
- **Address Space**: 16 EiB (2^64 bytes)
- **PTE Size**: 8 bytes (u64)
- **PTE Address**: `page_table_base + (index * 8)`

## Translation Modes

### Single-Level (Flat) Page Table

A simple flat array mapping page numbers directly to page pointers.

**Structure (32-bit physical):**
- Array of 2^48 entries (sparse, allocated on demand)
- Each entry: 4 bytes (u32) page pointer
- Entry address: `page_table_base + (page_num * 4)`

**Structure (64-bit physical):**
- Array of 2^48 entries (sparse, allocated on demand)
- Each entry: 8 bytes (u64) page pointer
- Entry address: `page_table_base + (page_num * 8)`

**Translation Algorithm:**
```
page_num = vaddr >> 16
page_pointer = page_table[page_num] // Load 4-byte or 8-byte entry

// 64-bit physical addressing
sec_idx = page_pointer & 0xFFFF
page_base_low48 = page_pointer >> 16
sec_entry = security_directory[sec_idx] // Load security entry
page_base_top16 = sec_entry.addr_top
phys_page_base = (page_base_top16 << 48) | page_base_low48

// 32-bit physical addressing
sec_idx = page_pointer & 0xFF
page_base_low24 = page_pointer >> 8
sec_entry = security_directory[sec_idx] // Load security entry
page_base_top8 = sec_entry.addr_top
phys_page_base = (page_base_top8 << 24) | page_base_low24

// Final address
page_offset = vaddr & 0xFFFF
phys_addr = phys_page_base + page_offset
```

### Multi-Level (Hierarchical) Page Table

A 3-level page table structure for sparse address spaces.

**Structure (32-bit physical):**
- Level 3 (top): 2^16 entries, each pointing to a Level 2 table (4 bytes/u32)
- Level 2: 2^16 entries, each pointing to a Level 1 table (4 bytes/u32)
- Level 1 (leaf): 2^16 entries, each containing a 4-byte page pointer
- All entries are 4 bytes (u32)

**Structure (64-bit physical):**
- Level 3 (top): 2^16 entries, each pointing to a Level 2 table (8 bytes/u64)
- Level 2: 2^16 entries, each pointing to a Level 1 table (8 bytes/u64)
- Level 1 (leaf): 2^16 entries, each containing an 8-byte page pointer
- All entries are 8 bytes (u64)

**Translation Algorithm:**
```
l3_idx = (vaddr >> 48) & 0xFFFF
l2_table = l3_table[l3_idx]

l2_idx = (vaddr >> 32) & 0xFFFF
l1_table = l2_table[l2_idx]

l1_idx = (vaddr >> 16) & 0xFFFF
page_pointer = l1_table[l1_idx] // Load 4-byte or 8-byte entry

// Final address translation is same as single-level
// ... (see single-level translation algorithm)
```

## Implementation-Specific Details

All implementations now require access to a `security_directory_base` in addition to the `page_table_base`.

### rift (RISC-V to WebAssembly)

**Location:** `rift/src/lib.rs`

**API:**
- Mappers now accept `security_directory_base`.
- Uses `Opts.mapper` callback for custom translation.

### r52x (RISC-V to x86-64)

**Location:** `r52x/src/lib.rs`

**API:**
- Mappers now accept `security_directory_base`.
- `MapperCallback` type for custom mppers.

### speet (RISC-V to WebAssembly Recompiler)

**Location:** `speet/crates/speet-riscv/src/lib.rs`

**API:**
- Mappers now accept `security_directory_base`.
- `set_mapper_callback()` method on `RiscVRecompiler`.

### vane (RISC-V JIT with JavaScript/WASM)

**Location:** `vane/crates/vane-jit/src/lib.rs`

**API:**
- `Mem` struct now holds `shared_security_directory_vaddr`.
- `translate_shared*` and `generate_*_js` methods updated.

## Page Table Entry (PTE) Format

### 32-bit Physical Addressing
```
Entry (4 bytes / 32 bits):
├─ Physical Page Base Address (lower 24 bits): bits [31:8]
└─ Security Directory Index: bits [7:0]
```

### 64-bit Physical Addressing
```
Entry (8 bytes / 64 bits):
├─ Physical Page Base Address (lower 48 bits): bits [63:16]
└─ Security Directory Index: bits [15:0]
```

## Security Directory

A new table containing security metadata for each page.

**Security Entry Format (32-bit physical):**
```
Entry (8 bytes / 64 bits):
├─ Physical Page Base Address (top 8 bits): bits [63:56]
├─ CFI Bitmask: bits [55:3]
├─ Reserved: bits [2:1]
└─ Read/Write/Execute Flags: bit [0]
```

**Security Entry Format (64-bit physical):**
```
Entry (8 bytes / 64 bits):
├─ Physical Page Base Address (top 16 bits): bits [63:48]
├─ CFI Bitmask: bits [47:3]
├─ Reserved: bits [2:1]
└─ Read/Write/Execute Flags: bit [0]
```

## Performance Characteristics

Lookup cost now includes an extra memory load to access the security directory.

## Security Considerations

1. **Bounds Checking**: Still required for all memory accesses.
2. **Page Permissions**: Now enforced via the Security Directory.
3. **Control Flow**: CFI metadata enables fine-grained control flow protection.
4. **Isolation**: Page tables continue to provide address space isolation.
5. **Nested Safety**: vane's nested mode keeps both page tables and the security directory in controlled virtual space.
