# XenevaOS Advanced Topics & Deep Dives
## Extended Technical Reference

---

## Table of Contents
1. [Interrupt & Exception Handling Details](#interrupt--exception-handling-details)
2. [Page Fault Resolution](#page-fault-resolution)
3. [Context Switching Mechanism](#context-switching-mechanism)
4. [Memory Protection](#memory-protection)
5. [Multiprocessor Support](#multiprocessor-support)
6. [Future Roadmap](#future-roadmap)

---

## Interrupt & Exception Handling Details

### Interrupt Descriptor Table (IDT) - x86-64

The IDT is a table that maps interrupt/exception vectors to handler functions.

**Structure**:
```
IDT
┌─────────────────────────────────┐
│ Vector 0: Divide by Zero        │ → Handler: divide_by_zero_handler
├─────────────────────────────────┤
│ Vector 1: Debug                 │ → Handler: debug_handler
├─────────────────────────────────┤
│ Vector 2: NMI                   │ → Handler: nmi_handler
├─────────────────────────────────┤
│ Vector 3: Breakpoint            │ → Handler: breakpoint_handler
├─────────────────────────────────┤
│ Vector 4: Overflow              │ → Handler: overflow_handler
├─────────────────────────────────┤
│ Vector 5: Bound Check           │ → Handler: bound_handler
├─────────────────────────────────┤
│ Vector 6: Invalid Opcode        │ → Handler: invalid_opcode_handler
├─────────────────────────────────┤
│ Vector 7: Device Not Available  │ → Handler: device_not_available_handler
├─────────────────────────────────┤
│ Vector 8: Double Fault          │ → Handler: double_fault_handler
├─────────────────────────────────┤
│ ...                             │
├─────────────────────────────────┤
│ Vector 14: Page Fault           │ → Handler: page_fault_handler
├─────────────────────────────────┤
│ Vector 32: IRQ0 (Timer)         │ → Handler: timer_interrupt_handler
├─────────────────────────────────┤
│ Vector 33: IRQ1 (Keyboard)      │ → Handler: keyboard_interrupt_handler
├─────────────────────────────────┤
│ ...                             │
└─────────────────────────────────┘
```

**IDT Entry Format** (64-bit):

```
Bits 0-15:   Offset low        (lower 16 bits of handler address)
Bits 16-31:  Segment selector  (kernel code segment)
Bits 32-34:  IST               (Interrupt Stack Table index)
Bits 35-39:  Reserved
Bits 40-43:  Gate type         (0xE = Interrupt, 0xF = Trap)
Bit 44:      Storage segment   (must be 0 for exceptions)
Bits 45-46:  DPL               (Descriptor Privilege Level)
Bit 47:      Present           (entry valid)
Bits 48-63:  Offset middle     (middle 16 bits)
Bits 64-95:  Offset high       (upper 32 bits)
Bits 96-127: Reserved
```

### Exception Handling Flow

**Step 1: Exception Occurs**
```c
mov rax, [0x0]  // Attempt to dereference NULL pointer
                // CPU detects invalid access
```

**Step 2: CPU Action**
```
1. Save current state:
   - RIP (instruction pointer)
   - RFLAGS (CPU flags)
   - RSP (stack pointer) - may switch to kernel stack
   
2. Look up vector in IDT:
   - Exception type determines vector
   - NULL pointer dereference → Vector 14 (Page Fault)
   
3. Load handler address from IDT entry
4. Change to kernel mode (CPL = 0)
5. Push state onto stack
6. Jump to handler
```

**Step 3: Exception Handler**

```c
void page_fault_handler(uint64_t error_code) {
    // Get address that caused fault
    uint64_t fault_address = read_cr2();
    
    // Determine cause
    if (error_code & 0x1) {
        // Protection violation
        if (error_code & 0x2) {
            // Write to read-only page
            kill_process(SIGSEGV);
        }
    } else {
        // Page not present
        if (is_valid_lazy_page(fault_address)) {
            // Allocate page on demand
            allocate_and_map_page(fault_address);
            // Return to retry instruction
        } else if (is_valid_stack_guard(fault_address)) {
            // Stack overflow
            kill_process(SIGSEGV);
        } else {
            // Invalid access
            kill_process(SIGSEGV);
        }
    }
}
```

**Step 4: Return from Handler**
```
1. IRET instruction (return from exception)
2. CPU restores state:
   - RIP (resume execution)
   - RFLAGS
   - RSP
3. Return to user mode or kernel code
```

### Interrupt Request (IRQ) Handling

**IRQ Timeline**:

```
Time →

Hardware Event
(disk read completes)
    │
    └─> Assert IRQ on bus (e.g., IRQ3)
        │
        └─> APIC (Advanced Programmable Interrupt Controller)
            receives signal
            │
            └─> Converts to vector number (32 + 3 = 35)
                │
                └─> Sends to CPU
                    │
                    └─> If interrupts enabled (IF flag):
                        │
                        └─> IDT lookup (vector 35)
                            │
                            └─> Jump to disk_interrupt_handler
                                │
                                ├─> Read device status
                                ├─> Copy data from device
                                ├─> Wake waiting process
                                ├─> EOI (End Of Interrupt) signal
                                │
                                └─> IRET (return from interrupt)
                                    │
                                    └─> Resume interrupted code
```

---

## Page Fault Resolution

### Page Fault Exception (Vector 14)

**Error Code Format**:
```
Bit 0: Present     (0 = page not in memory, 1 = protection violation)
Bit 1: Write       (0 = read fault, 1 = write fault)
Bit 2: User        (0 = kernel fault, 1 = user fault)
Bit 3: Reserved    (CPU sets on reserved bit violation)
Bit 4: Instruction (1 = instruction fetch fault)
Bits 5-63: Reserved
```

### Fault Handling Algorithm

**Case 1: Demand-Paged Memory (Lazy Allocation)**

```
Scenario: Application accesses heap that hasn't been allocated yet

1. Page fault on address 0x00700000 (error = 0)
   ├─> Page not present, not protection violation
   ├─> Check if address is in valid range:
   │   └─> Is 0x00700000 < heap_limit? YES
   │
   ├─> Allocate physical page
   │   └─> Find free page frame (bitmap)
   │   └─> Mark as allocated
   │
   ├─> Map to virtual address
   │   └─> Get page table entry for 0x00700000
   │   └─> Set present bit
   │   └─> Set address to physical page
   │   └─> Set permissions (user-mode readable/writable)
   │
   ├─> Invalidate TLB entry (optional, auto on IRET)
   │
   └─> Return from exception
       └─> Retry instruction that faulted
       └─> Page now accessible
```

**Case 2: Copy-On-Write (Optimization)**

```
Scenario: Child process shares parent's pages until it writes

Initial State:
┌─────────────────────────────────┐
│ Physical Memory Page P1         │ (contains data)
└─────────────────────────────────┘
      ▲                    ▲
      │                    │
      │ (both map here)    │
      │                    │
Parent VAddr          Child VAddr
0x00400000            0x00400000
(Read-Only)           (Read-Only)

Parent writes to page:
  mov qword [0x00400000], rax  // Fault!
  
Page Fault Handler:
  1. Is page marked copy-on-write?
     └─> YES
     
  2. Allocate new physical page P2
  
  3. Copy data from P1 to P2
  
  4. Update parent's page table:
     └─> Map 0x00400000 to P2
     └─> Set writable
     
  5. Return from exception
  
  6. Retry write instruction
  
Result:
┌─────────────────┐         ┌──────────────────┐
│ Physical Page P1│         │ Physical Page P2 │
│ (parent's copy) │         │ (child's copy)   │
└──────────┬──────┘         └─────────┬────────┘
           │                          │
           └──────────────┬───────────┘
                          │
         Parent can now modify P2 independently
```

**Case 3: Swap (Virtual Memory Extends to Disk)**

```
Scenario: Page swapped to disk because physical memory full

Page Fault Handler:
  1. Is page marked as swapped?
     └─> Check swap flag in page table entry
     └─> Get swap file location/offset
     
  2. Find physical page to evict
     └─> LRU (Least Recently Used) algorithm
     └─> Mark victim as swapped
     └─> Write to swap file
     
  3. Load faulting page from swap
     └─> Read from swap file
     └─> Put in allocated physical page
     
  4. Update page table
     └─> Clear swapped flag
     └─> Set present bit
     └─> Point to new physical page
     
  5. Return from exception
```

---

## Context Switching Mechanism

### Why Context Switch?

Modern CPUs are single-threaded (per core). Multiple threads/processes need CPU time, so scheduler must:
1. Stop running thread
2. Save its state
3. Load another thread's state
4. Resume execution

### Context Switch Components

**1. Thread Registers to Save/Restore**:

x86-64:
```
RAX, RBX, RCX, RDX, RSI, RDI, RBP
R8-R15
RSP (stack pointer)
RIP (instruction pointer)
RFLAGS (CPU flags)
```

ARM64:
```
X0-X30 (general purpose)
SP (stack pointer)
PC (program counter)
PSTATE (processor state)
```

**2. Context Structure**:

```c
typedef struct {
    // x86-64 general purpose registers
    uint64_t rax, rbx, rcx, rdx;
    uint64_t rsi, rdi, rbp;
    uint64_t r8, r9, r10, r11, r12, r13, r14, r15;
    
    // Special registers
    uint64_t rsp;  // Stack pointer
    uint64_t rip;  // Instruction pointer
    uint64_t rflags; // CPU flags
    
    // Control registers
    uint64_t cr3;  // Page table base
    
    // Segment registers (less important in 64-bit)
    uint16_t cs, ds, es, ss;
    
    // FPU/SSE state (if applicable)
    uint8_t fpu_state[512];  // x87 FPU state
    __m256 xmm[16];          // SSE/AVX registers
} ThreadContext;
```

### Scheduling Algorithm

**Time-Slicing with Priority**:

```
Ready Queue (sorted by priority):

Priority 10 ┌─────────────────────┐
            │ Thread A (20ms used) │ ← Currently running
            │ RIP = 0x600150      │
            │ RSP = 0x7FF00000    │
            └─────────────────────┘

Priority 8  ┌─────────────────────┐
            │ Thread B (15ms used) │ ← Next highest priority
            │ RIP = 0x700050      │
            │ RSP = 0x7FF20000    │
            └─────────────────────┘

Priority 5  ┌─────────────────────┐
            │ Thread C (5ms used)  │ ← Lower priority
            │ RIP = 0x500100      │
            │ RSP = 0x7FF40000    │
            └─────────────────────┘

Scheduler Tick (every 20ms):
1. Time slice expired for Thread A
2. Save Thread A context
3. Remove from ready queue
4. Select highest priority ready thread (B)
5. Load Thread B context
6. Resume Thread B
```

### Assembly-Level Context Switch (x86-64)

```asm
; Save current thread context
; Register state automatically saved by CPU on interrupt

; Scheduler runs (kernel code)
mov rax, next_thread_ptr     ; RAX = pointer to next thread's context

; Restore next thread context
mov rsp, [rax + offset_rsp]  ; Load stack pointer
mov r8,  [rax + offset_r8]   ; Restore R8
mov r9,  [rax + offset_r9]   ; Restore R9
... (restore other registers)

mov cr3, [rax + offset_cr3]  ; Load page table (switches address spaces!)
; TLB (Translation Lookaside Buffer) automatically flushed

; Return to user mode
iretq  ; or ret depending on context
```

---

## Memory Protection

### Protection Mechanisms

**1. Virtual Memory Isolation**

```
Process A                          Process B
Virtual 0x400000 ─┐              Virtual 0x400000 ─┐
                  │              Same address!      │
                  ▼                                 ▼
              Physical 0x1000000                Physical 0x2000000
                                 (Different physical pages)

Result: Each process has private memory at same virtual address
        Cannot access other process's memory
```

**2. User/Kernel Mode Boundary**

```
User Mode Code                     Kernel Mode Code
mov rax, [0xFFFFFF10...]  ✓ OK   mov rax, [0x1000000]      ✓ OK
(user memory access)             (any memory access)

mov rax, [0xFFFFFF10...]  ✗ FAULT (kernel memory from user)
Try to access kernel memory       → Page Fault Exception
via normal addressing             → CPU switches to kernel mode
                                  → Kernel handles exception
                                  → Likely kills process
```

**3. Page-Level Permissions**

```
Page Table Entry:
┌──────────────────────────────────────┐
│ ... | NX | Write | User | Present |  │
│     │ 0  │  0   │  1   │   1    │    │ = User-readable, not writable, executable
│     │ 0  │  1   │  1   │   1    │    │ = User-readable, writable, executable
│     │ 1  │  0   │  1   │   1    │    │ = User-readable, not writable, NOT executable
│     │ 1  │  1   │  1   │   1    │    │ = User-readable, writable, NOT executable
└──────────────────────────────────────┘

NX (No-Execute) Bit Prevents:
- Buffer overflow exploits (can't execute injected code)
- Return-to-libc attacks (if data is marked non-executable)
```

**4. Protection Key (PKU) - Advanced**

```
Modern CPUs allow 16 protection keys per process:
Each page can be associated with a protection key
Allows fine-grained permissions without page table changes

Use Case: Kernel Data Separation
─────────────────────────────────
Virtual Address 0xFFFFC00000000000
│
├─> Kernel Code Page (PK = 0, kernel-only)
│
├─> Kernel Data Page (PK = 1, protected)
│   └─> Can set user-mode access with wrpkru instruction
│   └─> Allows selective user access to kernel data
```

---

## Multiprocessor Support

### Multiprocessor Architecture

**Physical Layout**:
```
┌──────────────────────────────────┐
│        Main Memory (RAM)         │
│  (Shared by all processors)      │
└────────────┬─────────────────────┘
             │
        ┌────┴────┬────────┬────────┐
        │          │        │        │
    ┌───▼──┐  ┌───▼──┐ ┌───▼──┐ ┌──▼───┐
    │ Core0│  │ Core1│ │ Core2│ │ Core3│
    │ CPU0 │  │ CPU1 │ │ CPU2 │ │ CPU3 │
    │      │  │      │ │      │ │      │
    │ L1-D │  │ L1-D │ │ L1-D │ │ L1-D │ (L1 cache per core)
    │ L1-I │  │ L1-I │ │ L1-I │ │ L1-I │
    │      │  │      │ │      │ │      │
    │ L2   │  │ L2   │ │ L2   │ │ L2   │ (L2 cache per core)
    └───┬──┘  └───┬──┘ └───┬──┘ └──┬───┘
        │         │        │       │
        └─────┬───┴────┬───┴───┬───┘
              │        │       │
          ┌───▼────────▼───────▼───┐
          │   L3 Cache (shared)    │
          └────────────────────────┘
              │
        ┌─────▼─────┐
        │ Front-side│
        │   Bus     │  (connects to memory controller)
        └───────────┘
```

### Synchronization Challenges

**Problem: Cache Coherency**

```
Initial: memory[0] = 0

Core 0:                        Core 1:
load rax, [0]  → rax=0        (rax still has old value)
mov [0], 5                     load rax, [0]  → Should be 5?

Without synchronization:
Core 0's L1 cache: [0] = 5
Core 1's L1 cache: [0] = 0    (stale value!)

Solution: MESI Protocol
───────────────────────
Modified:   Data changed locally, not yet in main memory
Exclusive:  Only this core has the data
Shared:     Multiple cores have the data
Invalid:    Core's copy is outdated

When Core 0 writes [0] = 5:
1. Core 0's L1: state = Modified
2. Core 0 broadcasts invalidation
3. Core 1's L1: state = Invalid
4. Core 1 re-reads, gets new value
```

### XenevaOS Multiprocessor Support

**Current Status**: Framework in place, scheduler not fully ready

**AP (Application Processor) Code**:

```c
// Boot-time code to initialize secondary CPUs
void ap_startup(void) {
    // Each AP executes this
    
    // Initialize local APIC
    setup_apic();
    
    // Get processor-specific memory areas
    // (stack, GDT, etc.)
    
    // Enter kernel code
    kernel_main_ap();
}

// In XenevaOS, AP code pointer passed via KernelBootInfo:
// kbi->apcode = address of AP initialization code
```

**Spinlock Example**:

```c
typedef struct {
    volatile int locked;  // 0 = unlocked, 1 = locked
} Spinlock;

void spinlock_acquire(Spinlock *lock) {
    while (atomic_cmpxchg(&lock->locked, 0, 1) != 0) {
        // Keep spinning until we acquire
        asm("pause");  // Hint to CPU: we're in busy-wait loop
    }
}

void spinlock_release(Spinlock *lock) {
    atomic_write(&lock->locked, 0);
}

// Usage:
Spinlock lock = {0};
spinlock_acquire(&lock);
// Critical section (only one thread at a time)
spinlock_release(&lock);
```

### Load Balancing

**Goal**: Distribute work evenly across cores

```
Scheduler Decision:
1. Check load on each core (number of ready threads)
2. Assign new thread to least-loaded core
3. Move threads between cores if imbalance detected

Example:
Core 0: [Thread_A] [Thread_B]  (2 threads)
Core 1: [Thread_C] [Thread_D] [Thread_E]  (3 threads)
Core 2: [Thread_F]  (1 thread)

Load: Core0=2, Core1=3, Core2=1
Action: Move a thread from Core1 to Core2
Result: Core0=2, Core1=2, Core2=2 (balanced)
```

---

## Future Roadmap

### Planned Features

**RISC-V Support**:
- Architecture-agnostic kernel already in place
- RISC-V HAL needs implementation
- Target for future releases

**User-Mode Drivers**:
- Isolate driver crashes from kernel
- Easier debugging and updates
- IPC overhead tradeoff

**Advanced Memory Features**:
- NUMA (Non-Uniform Memory Access) support
- Huge pages (2MB, 1GB)
- Memory compression (swap to compressed memory)

**Security Enhancements**:
- Mandatory Access Control (MAC)
- Capability-based security
- SELinux-like policies

**3D Graphics & XR**:
- Native 3D desktop environment
- VR/AR application support
- GPU acceleration

**Additional Filesystems**:
- ELF binary support
- Advanced filesystem features (snapshots, compression)
- Network filesystem (NFS)

**Advanced Networking**:
- IPv6 support
- WiFi drivers
- Network security (IPsec, TLS)

### Research Areas

**Memory Optimization**:
- Investigate reduced abstraction layers
- Memory tagging extensions (MTE)
- Pointer authentication

**Performance**:
- Lock-free data structures
- SIMD optimizations
- Kernel preemption tuning

**Reliability**:
- Fault tolerance mechanisms
- Self-healing capabilities
- Monitoring and telemetry

**Application Domains**:
- Embedded systems
- High-performance computing
- Edge computing
- IoT devices

---

## Conclusion

XenevaOS demonstrates sophisticated operating system design with:
- Clean architecture suitable for research
- Modern hardware utilization
- Extensible framework for future features
- Complete implementation stack

This foundation enables exploration of:
- Alternative OS designs
- New security models
- Performance optimization techniques
- Domain-specific computing solutions

The project serves as both a learning resource and a platform for advancing operating system technology.

---

**Document Version**: 1.0  
**Last Updated**: January 2026  
**Status**: Early Development (Foundation Complete)
