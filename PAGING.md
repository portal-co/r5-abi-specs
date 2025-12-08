# RISC-V Backend Paging System Specification

This document specifies the unified paging system used across all RISC-V backend implementations (rift, r52x, speet, vane).

## Overview

All RISC-V backends share a common 64KB page-based virtual memory system with support for both single-level and multi-level page tables. This provides a consistent interface for memory address translation across different compilation targets (WebAssembly, x86-64, JavaScript).

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
- **Entry Size**: 4 bytes (u32)
- **Entry Address**: `page_table_base + (index * 4)`

### 64-bit Physical Addressing
- **Address Space**: 16 EiB (2^64 bytes)
- **Entry Size**: 8 bytes (u64)
- **Entry Address**: `page_table_base + (index * 8)`

## Translation Modes

### Single-Level (Flat) Page Table

A simple flat array mapping page numbers directly to physical pages.

**Structure (32-bit physical):**
- Array of 2^48 entries (sparse, allocated on demand)
- Each entry: 4 bytes (u32) containing physical page base address
- Entry address: `page_table_base + (page_num * 4)`

**Structure (64-bit physical):**
- Array of 2^48 entries (sparse, allocated on demand)
- Each entry: 8 bytes (u64) containing physical page base address
- Entry address: `page_table_base + (page_num * 8)`

**Translation Algorithm:**
```
page_num = vaddr >> 16
phys_page = page_table[page_num]  // Load 4-byte or 8-byte entry
page_offset = vaddr & 0xFFFF
phys_addr = phys_page + page_offset
```

### Multi-Level (Hierarchical) Page Table

A 3-level page table structure for sparse address spaces.

**Structure (32-bit physical):**
- Level 3 (top): 2^16 entries, each pointing to a Level 2 table (4 bytes/u32)
- Level 2: 2^16 entries, each pointing to a Level 1 table (4 bytes/u32)
- Level 1 (leaf): 2^16 entries, each containing physical page base (4 bytes/u32)
- All entries are 4 bytes (u32)

**Structure (64-bit physical):**
- Level 3 (top): 2^16 entries, each pointing to a Level 2 table (8 bytes/u64)
- Level 2: 2^16 entries, each pointing to a Level 1 table (8 bytes/u64)
- Level 1 (leaf): 2^16 entries, each containing physical page base (8 bytes/u64)
- All entries are 8 bytes (u64)

**Translation Algorithm:**
```
l3_idx = (vaddr >> 48) & 0xFFFF
l2_table = l3_table[l3_idx]  // Load 4-byte or 8-byte entry

l2_idx = (vaddr >> 32) & 0xFFFF
l1_table = l2_table[l2_idx]  // Load 4-byte or 8-byte entry

l1_idx = (vaddr >> 16) & 0xFFFF
phys_page = l1_table[l1_idx]  // Load 4-byte or 8-byte entry

page_offset = vaddr & 0xFFFF
phys_addr = phys_page + page_offset
```

## Implementation-Specific Details

### rift (RISC-V to WebAssembly)

**Location:** `rift/src/lib.rs`

**API:**
- `standard_page_table_mapper()` - generates WASM code for single-level
- `standard_page_table_mapper_32()` - 32-bit physical addressing variant
- `multilevel_page_table_mapper()` - generates WASM code for multi-level
- `multilevel_page_table_mapper_32()` - 32-bit physical addressing variant
- Uses `Opts.mapper` callback for custom translation

**Usage:**
```rust
// 64-bit physical addresses (default)
let phys_addr = standard_page_table_mapper(
    module, function, block, vaddr,
    page_table_base, memory
);

// 32-bit physical addresses
let phys_addr = standard_page_table_mapper_32(
    module, function, block, vaddr,
    page_table_base, memory
);
```

### r52x (RISC-V to x86-64)

**Location:** `r52x/src/lib.rs`

**API:**
- `standard_page_table_mapper()` - emits x86-64 assembly
- `multilevel_page_table_mapper()` - emits x86-64 for 3-level
- `MapperCallback` type for custom mappers

**Register Convention:**
- Input: RAX contains virtual address
- Output: RAX contains physical address
- Clobbered: RBX, RCX, RDX (multi-level only)

**Usage:**
```rust
let mut mapper = |w, cfg| {
    standard_page_table_mapper(w, cfg, pt_base)
};
x86ify(writer, cfg, input, entry, labels, Some(&mut mapper))?;
```

### speet (RISC-V to WebAssembly Recompiler)

**Location:** `speet/crates/speet-riscv/src/lib.rs`

**API:**
- `standard_page_table_mapper()` - emits WASM instructions (64-bit physical)
- `standard_page_table_mapper_32()` - emits WASM instructions (32-bit physical)
- `multilevel_page_table_mapper()` - emits WASM for 3-level (64-bit physical)
- `multilevel_page_table_mapper_32()` - emits WASM for 3-level (32-bit physical)
- `set_mapper_callback()` method on `RiscVRecompiler`
- Uses unified `CallbackContext` for all callbacks

**Stack Convention:**
- Virtual address must be saved to local 66 before calling
- Physical address left on stack after calling

**Usage:**
```rust
// 64-bit physical addresses
recompiler.set_mapper_callback(&mut |ctx| {
    standard_page_table_mapper(ctx, pt_base, 0, true)
});

// 32-bit physical addresses
recompiler.set_mapper_callback(&mut |ctx| {
    standard_page_table_mapper_32(ctx, pt_base, 0, true)
});
```

### vane (RISC-V JIT with JavaScript/WASM)

**Location:** `vane/crates/vane-jit/src/lib.rs`

**Special Features:**
- Three modes: Legacy, Shared, Both
- **Nested Paging**: Shared page table stored IN legacy virtual memory
- JavaScript code generation for inline translation

**Modes:**

1. **Legacy Mode** (default)
   - On-demand page allocation via BTreeMap
   - No explicit page table
   
2. **Shared Mode**
   - Explicit page table compatible with other backends
   - Can be inlined into JavaScript

3. **Both Mode** (Nested Paging)
   - Page table stored at virtual address in legacy system
   - Two-level translation:
     - Shared table provides address mapping
     - Legacy system handles table storage and allocation
   - All table accesses go through `get_page()`

**API:**
- `translate_shared()` - single-level nested translation (64-bit physical)
- `translate_shared_32()` - single-level nested translation (32-bit physical)
- `translate_shared_multilevel()` - 3-level nested translation (64-bit physical)
- `translate_shared_multilevel_32()` - 3-level nested translation (32-bit physical)
- `generate_shared_paging_js()` - inline JS for single-level (64-bit physical)
- `generate_shared_paging_js_32()` - inline JS for single-level (32-bit physical)
- `generate_multilevel_paging_js()` - inline JS for multi-level (64-bit physical)
- `generate_multilevel_paging_js_32()` - inline JS for multi-level (32-bit physical)

**Nested Architecture:**
```
Virtual Address (in shared table)
      ↓
Shared Page Table Lookup
  - Table at vaddr in legacy space
  - Accessed via get_page()
      ↓
Intermediate Address
      ↓
Legacy Page Allocation
  - On-demand allocation
      ↓
Physical Memory
```

## Page Table Entry Format

All implementations support two entry formats:

### 32-bit Physical Addressing
```
Entry (4 bytes / 32 bits):
├─ Physical Page Base Address: bits [31:16] - aligned to 64KB
└─ Reserved/Flags: bits [15:0] - currently unused (must be 0)
```
**Address Space:** Up to 4 GiB (2^32 bytes)

### 64-bit Physical Addressing
```
Entry (8 bytes / 64 bits):
├─ Physical Page Base Address: bits [63:16] - aligned to 64KB
└─ Reserved/Flags: bits [15:0] - currently unused (must be 0)
```
**Address Space:** Up to 16 EiB (2^64 bytes)

**Note:** The lower 16 bits are reserved for future use (permission flags, etc.) and must be zero in current implementations.

## Performance Characteristics

### Single-Level (32-bit physical)
- **Memory**: 2^48 * 4 bytes = 1 PiB (sparse, only allocated pages exist)
- **Lookup Cost**: 1 memory load (4 bytes)
- **Best For**: Dense address spaces, simple mappings, memory-constrained systems

### Single-Level (64-bit physical)
- **Memory**: 2^48 * 8 bytes = 2 PiB (sparse, only allocated pages exist)
- **Lookup Cost**: 1 memory load (8 bytes)
- **Best For**: Dense address spaces, simple mappings, large physical memory

### Multi-Level (32-bit physical)
- **Memory**: Depends on address space utilization (typically much less)
- **Lookup Cost**: 3 memory loads (4 bytes each)
- **Best For**: Sparse address spaces, memory-constrained systems

### Multi-Level (64-bit physical)
- **Memory**: Depends on address space utilization (typically much less)
- **Lookup Cost**: 3 memory loads (8 bytes each)
- **Best For**: Sparse address spaces, large physical memory

### Nested (vane Both mode)
- **Memory**: Same as multi-level, but allocated via legacy system
- **Lookup Cost**: 3-7 memory loads (4 or 8 bytes each, depending on level + legacy overhead)
- **Best For**: Controlled mapping within virtual environment

## Security Considerations

1. **Bounds Checking**: Implementations should validate addresses
2. **Page Permissions**: Reserved bits in entries can be used for RWX flags
3. **Isolation**: Each page table provides address space isolation
4. **Nested Safety**: vane's nested mode keeps tables in controlled virtual space

## Future Extensions

- **Permission Bits**: Use reserved entry bits for read/write/execute flags
- **Page Table Walking**: Hardware-style TLB miss handlers
- **Huge Pages**: 1MB or larger pages for reduced table overhead
- **ASID Support**: Address Space IDentifier for multi-process support

## Compatibility Notes

All implementations are designed to be binary-compatible at the page table level. Page tables created for one backend can be used with another (assuming appropriate address space sizes and memory models).

## See Also

- rift: WebAssembly compilation with `portal-pc-waffle`
- r52x: x86-64 native code generation with `portal-solutions-asm-x86-64`
- speet: WebAssembly recompilation with `yecta` reactor
- vane: JavaScript JIT with hybrid paging modes
