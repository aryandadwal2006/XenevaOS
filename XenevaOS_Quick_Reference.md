# XenevaOS Architecture & Component Reference Guide
## Quick Reference & Visual Mapping

---

## Project Structure

```
XenevaOS/
├── Aurora.sln              (Visual Studio Solution)
├── readme.md               (Main README)
├── Docs/                   (Documentation)
│   ├── Introduction.md
│   ├── XELoader.md
│   ├── BuildInstructions.md
│   ├── VMSetup.md
│   ├── Kernel/
│   │   ├── AboutKernel.md
│   │   ├── BootProcess.md
│   │   ├── MemoryMangement.md
│   │   ├── Drivers.md
│   │   ├── KernelServices.md
│   │   └── usb.md
│   └── Development/
│       ├── DriverDevelopment.md
│       └── ApplicationDevelopment.md
│
├── Boot/                   (Bootloader - UEFI x86-64)
├── BootAA64/              (Bootloader - UEFI/LittleBoot ARM64)
├── XELoader/              (Dynamic Linker/Loader)
│
├── Kernel/                (x86-64 Kernel Implementation)
│   ├── Mm/                (Memory Management)
│   ├── Mm/Pmm/            (Physical Memory Manager)
│   ├── Mm/Vmm/            (Virtual Memory Manager)
│   ├── Mm/Shm/            (Shared Memory Manager)
│   ├── Fs/                (File System)
│   ├── Drivers/           (Boot-time Drivers)
│   ├── Net/               (Networking)
│   ├── Sound/             (Audio)
│   ├── Ipc/               (Inter-Process Communication)
│   ├── Sync/              (Synchronization)
│   ├── Hal/               (Hardware Abstraction Layer)
│   ├── Serv/              (Services)
│   ├── x64/               (x86-64 Assembly & specific code)
│   ├── process.cpp        (Process Management)
│   ├── loader.cpp         (Executable Loading)
│   ├── pcie.cpp           (PCI Express)
│   ├── pe.cpp             (PE binary parsing)
│   └── ...
│
├── KernelAA64/            (ARM64 Kernel Implementation)
│   └── [Same structure as Kernel/ but for ARM64]
│
├── Drivers/               (Runtime Drivers)
│   ├── AHCI/              (SATA/AHCI Storage Driver)
│   ├── NVMe/              (NVMe Storage Driver)
│   ├── USB/               (USB 2.0 Driver)
│   ├── Usb3/              (USB 3.0/xHCI Driver)
│   ├── E1000/             (Intel Ethernet Driver)
│   ├── Sound/             (Intel HD Audio Driver)
│   ├── VMSvga/            (SVGA Graphics Driver)
│   ├── Virtio/            (Virtio Framework)
│   ├── vbox/              (VirtualBox Guest Additions)
│   ├── vmware/            (VMware Tools)
│   └── xHCI/              (USB 3.0 Host Controller)
│
├── Libs/                  (User-Space Libraries)
│   ├── XEClib/            (Xeneva C Library - POSIX-like APIs)
│   ├── Chitralekha/       (Graphics & Widget Library)
│   └── XECrt/             (C Runtime - startup, cleanup)
│
├── Process/               (User-Space Applications)
│   ├── Init/              (System Initialization)
│   ├── Namdapha/          (Desktop Environment)
│   ├── Deodhai/           (Window Manager)
│   ├── DeodhaiAudio/      (Audio Server)
│   ├── DeodhaiXR/         (XR/VR Framework)
│   ├── Terminal/          (Terminal Emulator)
│   ├── XEShell/           (Command Shell)
│   ├── XELnch/            (Application Launcher)
│   ├── Files/             (File Browser)
│   ├── AudioPlayer/       (Audio Player - Accent Player)
│   ├── Calculator/        (Calculator)
│   ├── Calender/          (Calendar)
│   ├── NETMngr/           (Network Manager)
│   ├── Control/           (Control Panel)
│   ├── cat/               (File viewer)
│   ├── ping/              (Network utility)
│   ├── nslook/            (DNS lookup)
│   ├── play/              (Audio playback)
│   ├── mp4plr/            (Video player)
│   ├── rm/                (File removal)
│   └── systray/           (System tray)
│
├── Acpica/                (ACPI Interface - third-party)
├── BaseHdr/               (Base Headers - kernel interfaces)
├── Build/                 (Build scripts and artifacts)
├── LittleBoot/            (ARM64 Lightweight bootloader)
└── Resources/             (Static resources, fonts, etc.)
```

---

## Kernel Subsystem Organization

```
┌────────────────────────────────────────────────────────┐
│              KERNEL ARCHITECTURE LAYERS                │
└────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  Processes/Threads/Scheduling (Serv, process.cpp)    │
│  - Process creation/termination                       │
│  - Thread management                                  │
│  - CPU scheduling                                     │
│  - Process state management                           │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  Memory Management (Mm/)                              │
│  - Physical Memory Manager (Pmm)                      │
│    * Page allocation/deallocation                     │
│    * Bitmap-based tracking                            │
│  - Virtual Memory Manager (Vmm)                       │
│    * Page tables                                      │
│    * Address space management                         │
│  - Shared Memory Manager (Shm)                        │
│    * Inter-process memory sharing                     │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  File System (Fs/)                                    │
│  - VFS (Virtual File System) abstraction              │
│  - Inode management                                   │
│  - Block I/O operations                               │
│  - File operations (read, write, seek)                │
│  - Directory navigation                               │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  Device Management                                    │
│  - Boot Drivers (Kernel/Drivers/)                     │
│    * Storage drivers (AHCI, NVMe)                     │
│    * Filesystem drivers                               │
│    * ACPI driver                                      │
│  - Runtime Drivers (Drivers/)                         │
│    * Network drivers                                  │
│    * Audio drivers                                    │
│    * Graphics drivers                                 │
│  - Device registration/enumeration                    │
│  - devfs (device filesystem)                          │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  IPC & Synchronization (Ipc/, Sync/)                 │
│  - Shared memory                                      │
│  - Pipes                                              │
│  - Sockets                                            │
│  - Spinlocks/Mutexes/Semaphores                       │
│  - Wait queues                                        │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  Interrupt/Exception Handling (Hal/)                  │
│  - IDT/Exception vector setup                         │
│  - Interrupt handlers                                 │
│  - Context switching                                  │
│  - Mode switching (user ↔ kernel)                     │
└──────────────────────────────────────────────────────┘
                         ↑
┌──────────────────────────────────────────────────────┐
│  Hardware Abstraction Layer (Hal/)                    │
│  - CPU operations                                     │
│  - Memory operations (paging, TLB)                    │
│  - Timer control                                      │
│  - Architecture-specific code                         │
│    * Kernel/x64/ (x86-64)                             │
│    * KernelAA64/ (ARM64)                              │
└──────────────────────────────────────────────────────┘
```

---

## Execution Flow: From Boot to Application

```
POWER ON
  │
  └─> UEFI Firmware
      - Initialize hardware
      - Locate bootloader
      - Load XNLDR (Xeneva Loader)
      
  └─> XNLDR (Bootloader)
      - Get memory map from UEFI
      - Load kernel executable
      - Load boot drivers
      - Create KernelBootInfo structure
      - Set up paging (higher half kernel)
      - Call kernel entry point
      
  └─> Aurora Kernel (x86-64/ARM64)
      
      1. Early Initialization
         - Initialize memory manager
         - Parse memory map
         - Initialize interrupt handlers
         - Set up framebuffer
         
      2. Core Subsystem Initialization
         - Initialize virtual memory manager
         - Initialize file system
         - Initialize process management
         
      3. Boot Driver Initialization
         - Initialize storage drivers (AHCI, NVMe)
         - Initialize filesystem driver
         - Initialize ACPI driver
         
      4. Device Enumeration & Runtime Drivers
         - Scan PCI devices (via ACPI)
         - Load matching drivers
         - Initialize network, audio, graphics
         
      5. Create Init Process
         - Create first user-space process
         - Load /init.exe
         - Switch to user mode
         
  └─> Init Process
      - Starts essential services:
        * Deodhai (window manager)
        * DeodhaiAudio (audio server)
        * Namdapha (desktop environment)
        * NETMngr (network manager)
      - Ready for user interaction
      
  └─> User Interaction
      - User launches applications
      - Shell interprets commands
      - Applications load via XELoader
      - Run with kernel services
```

---

## Data Flow: User Application to Hardware

```
User Application Code
  │
  │ Makes library call
  ├─> XEClib (e.g., fopen())
  │
  │ Calls system call
  ├─> syscall instruction
  │
  │ CPU Mode Switch: User → Kernel
  │
  ├─> Kernel Syscall Dispatcher
  │   ├─> Validates arguments
  │   ├─> Checks permissions
  │
  │   ├─> File System Handler
  │   │   ├─> VFS operations
  │   │   ├─> Inode lookup
  │   │   ├─> Block allocation
  │   │
  │   ├─> Filesystem Driver (FAT32, NTFS, etc.)
  │
  ├─> Device Driver (AHCI, NVMe, etc.)
  │   ├─> Translate to hardware commands
  │   ├─> Setup DMA transfers
  │   ├─> Trigger hardware
  │
  ├─> Physical Hardware
  │   ├─> Disk controller
  │   └─> Storage device
  │       (reads data from disk)
  │
  │ Hardware Interrupt
  │ (data transfer complete)
  │
  ├─> Interrupt Handler
  │   ├─> Copy data to kernel buffer
  │   ├─> Wake waiting process
  │
  │ CPU Mode Switch: Kernel → User
  │
  └─> Application receives result
```

---

## System Call Calling Convention

### x86-64

```
User Application                     Kernel Handler
─────────────────                    ───────────────

mov r12, 5                          Syscall number validated
mov r13, arg1                       Arguments extracted
mov r14, arg2                       from registers:
mov r15, arg3                       - r13, r14, r15, rdi
mov rdi, arg4                       - Stack (additional args)
syscall                             
                                    ┌──────────────────────┐
                                    │ Handler executes     │
                                    │ in kernel mode       │
                                    └──────────────────────┘
                                    
                                    mov rax, return_value
                                    return from handler
                                    
← ret rax                           Result in RAX register
```

### ARM64

```
User Application                     Kernel Handler
─────────────────                    ───────────────

mov x16, 23                         Syscall number validated
mov x0, arg1                        Arguments extracted
mov x1, arg2                        from x0-x5 registers
mov x2, arg3
mov x3, arg4
mov x4, arg5
mov x5, arg6
svc #0                              
                                    ┌──────────────────────┐
                                    │ Handler executes     │
                                    │ in kernel mode       │
                                    └──────────────────────┘
                                    
                                    mov x6, return_value
                                    return from handler
                                    
mov x0, x6                          Result in X6 register
ret                                 Return to user mode
```

---

## Memory Address Space Layout

### x86-64

```
Virtual Address          │  Contents                │  Access
─────────────────────────┼──────────────────────────┼─────────
0xFFFFFFFFFFFFFFFF       │  (invalid/reserved)      │  NONE
                         │                          │
0xFFFFF01000000000       │  MMIO Device Memory      │  K-RW
                         │  (device registers)      │
                         │                          │
0xFFFF800000000000       │  Linear Physical Map     │  K-RW
                         │  (entire RAM linear)     │  (P2V/V2P)
                         │                          │
0xFFFFC00000000000       │  Higher Half Kernel      │  K-RW
                         │  (kernel code/data)      │  Shared across
                         │  (kernel stack)          │  all processes
                         │                          │
0x7FFFFFFF FFFFFFFF      │  (gap)                   │
                         │                          │
                         │  User Stack              │  U-RW
                         │  (grows downward)        │
                         │  (guard page)            │
                         │                          │
                         │  User Heap               │  U-RW
                         │  (grows upward)          │
                         │                          │
0x0000600000000000       │  User Code/Data          │  U-RE
                         │  (default base address)  │  (executable)
                         │                          │
0x0000000000000000       │  (invalid for safety)    │  NONE
```

K = Kernel, U = User, R = Read, W = Write, E = Execute

### ARM64

Similar to x86-64 but with potential variations in exact addresses depending on page size (4KB typical).

---

## I/O Device Discovery & Driver Loading Process

```
1. ACPI Tables Loaded
   ↓
2. System Enumerates PCI Devices
   ├─> Bus 0, Device 1, Function 0
   │   Class: 0x01 (Storage)
   │   SubClass: 0x06 (SATA)
   │   ProgIf: 0x00 (AHCI)
   │
   ├─> Bus 0, Device 3, Function 0
   │   Class: 0x02 (Network)
   │   SubClass: 0x00 (Ethernet)
   │   Device ID: 0x8086:0x1539 (Intel E1000)
   │
   └─> Bus 1, Device 0, Function 0
       Class: 0x04 (Multimedia)
       SubClass: 0x03 (Audio)
       (Intel HD Audio)

3. Load Driver Configuration
   ├─> /etc/drivers.conf
   │   [ahci]
   │   path = /drivers/ahci.dll
   │   class = 0x01, subclass = 0x06, progif = 0x00
   │
   │   [e1000]
   │   path = /drivers/e1000.dll
   │   class = 0x02, subclass = 0x00, progif = *
   │
   └─> [audio]
       path = /drivers/audio.dll
       class = 0x04, subclass = 0x03, progif = *

4. Match & Load Drivers
   ├─> Device (0x01, 0x06, 0x00) matches [ahci]
   │   Load /drivers/ahci.dll
   │   Call AuDriverMain()
   │
   ├─> Device (0x02, 0x00, *) matches [e1000]
   │   Load /drivers/e1000.dll
   │   Call AuDriverMain()
   │
   └─> Device (0x04, 0x03, *) matches [audio]
       Load /drivers/audio.dll
       Call AuDriverMain()

5. Driver Initialization
   ├─> AHCI Driver
   │   ├─> Find device on PCI bus
   │   ├─> Get base address (BAR0)
   │   ├─> Map MMIO region
   │   ├─> Initialize controller
   │   ├─> Allocate MSI interrupt
   │   ├─> Register interrupt handler
   │
   ├─> E1000 Driver
   │   ├─> Find device on PCI bus
   │   ├─> Get base address
   │   ├─> Map MMIO region
   │   ├─> Initialize NIC
   │   ├─> Allocate interrupt
   │   └─> Register with network stack
   │
   └─> Audio Driver
       ├─> Find device on PCI bus
       ├─> Get resources
       ├─> Initialize audio controller
       └─> Register with audio server

6. Devices Ready for Use
   ├─> /dev/disk0/ahci1 (first SATA disk)
   ├─> /dev/net/e1000 (ethernet)
   └─> /dev/audio (audio device)
```

---

## Shared Memory IPC Pattern

```
Process A (Audio App)              Process B (DeodhaiAudio Server)
──────────────────────────────────────────────────────────────

Create shared memory:
AuCreateSHM(4096, key=0x5001)
    │
    └─> System allocates
        physical pages
        (e.g., 0x1000000)
        │
        Maps to ProcessA:
        virtual 0x500000
        │
        Returns SHM ID = 42

                                   Attach to shared memory:
                                   AuCreateSHM(same_key=0x5001)
                                       │
                                       System recognizes key
                                       │
                                       Maps SAME physical pages
                                       (0x1000000)
                                       │
                                       To ProcessB:
                                       virtual 0x700000

Now both processes see same data:

ProcessA: *(volatile int*)0x500000 = 44100  (sample rate)
          ↓
          Physical Memory: 0x1000000 = 44100
          ↑
ProcessB: sample_rate = *(volatile int*)0x700000  (same value)

Zero-copy IPC: No data copied, just virtual address mapped!
```

---

## Binary Execution Flow

```
user$ ./calculator.exe

Shell:
  1. Parse command line
  2. Find executable: /calculator.exe
  3. Call: AuCreateProcess()
  
Kernel:
  4. Allocate process structure
  5. Create virtual address space
  6. Load executable via loader
  
Loader (in process context):
  7. Read executable header
  8. Determine: static or dynamic?
  
IF STATIC:
  9. Map code sections
  10. Map data sections
  11. Jump to entry point
  
IF DYNAMIC:
  9. Load XELoader instead
  10. Pass original filename to XELoader
  11. Jump to XELoader entry point
  
XELoader (user-mode):
  12. Load main executable
  13. Parse executable
  14. Identify dependencies:
      ├─> graphics.dll
      ├─> crt.dll
      └─> math.dll
  
  15. Load each library
  16. Resolve symbols:
      ├─> draw_circle() → graphics.dll:0x700500
      ├─> malloc() → crt.dll:0x600F00
      └─> sin() → math.dll:0x701200
  
  17. Relocate addresses
  18. Update jump tables
  
  19. Extract entry point: 0x600100
  20. Jump to entry point
  
Application (now running):
  21. main() function executes
  22. Can call library functions
  23. Can call system calls
  24. Interact with devices via kernel
```

---

## Key Data Structures

### Process Structure

```c
typedef struct {
    uint32_t pid;                    // Process ID
    uint32_t parent_pid;             // Parent process ID
    AuPageTable* page_table;         // Virtual address space
    List<Thread*> threads;           // Threads in process
    int fd_table[256];               // Open file descriptors
    SignalHandler handlers[32];      // Signal handlers
    uint64_t memory_limit;           // Max memory allowed
    List<SharedMem*> shm_list;       // Attached shared memory
    ProcessState state;              // Running, blocked, zombie
    uint32_t uid;                    // User ID
    uint32_t gid;                    // Group ID
} Process;
```

### Thread Structure

```c
typedef struct {
    uint32_t tid;                    // Thread ID
    Process* parent_process;         // Owning process
    Registers cpu_state;             // Saved CPU state
    uint64_t stack_ptr;              // Stack pointer
    ThreadState state;               // Running, blocked, ready
    int priority;                    // Scheduling priority
    uint64_t cpu_time_used;          // Total CPU time
} Thread;
```

### Page Table Entry (x86-64)

```c
typedef struct {
    uint64_t present : 1;            // Page present in memory
    uint64_t writable : 1;           // Writable page
    uint64_t user_mode : 1;          // User-mode accessible
    uint64_t write_through : 1;      // Write-through cache
    uint64_t cache_disabled : 1;     // Caching disabled
    uint64_t accessed : 1;           // Page accessed
    uint64_t dirty : 1;              // Page modified
    uint64_t pat : 1;                // PAT (memory type)
    uint64_t global : 1;             // Global page (TLB)
    uint64_t ignored : 3;            // Available
    uint64_t address : 40;           // Physical address (>>12)
    uint64_t ignored2 : 11;          // Available
    uint64_t no_execute : 1;         // No-execute bit
} PageTableEntry;
```

---

## Quick Lookup: Common Operations

### Creating a Process

```c
// Create new process
uint32_t pid = AuCreateProcess();

// Load executable
AuProcessLoadExec(pid, "/app.exe", 0, NULL);

// Process now runs when scheduler switches to it
```

### File Operations

```c
// Open file
int fd = AuOpenFile("/path/to/file", O_RDONLY);

// Read from file
uint8_t buffer[1024];
uint64_t bytes_read = AuReadFile(fd, buffer, sizeof(buffer));

// Write to file
AuWriteFile(fd, data, size);

// Close file
AuCloseFile(fd);
```

### Shared Memory

```c
// Create shared memory
uint16_t shm_id = AuCreateSHM(4096, key=0x1234);

// Get pointer to shared memory
void *ptr = AuSHMObtainMem(shm_id);

// Use like normal memory
int *shared_int = (int*)ptr;
*shared_int = 42;

// Detach when done
AuDetachSHM(shm_id);
```

### Graphics

```c
#include <chitralekha.h>

// Create window
window_t *wnd = create_window("My App", 640, 480);

// Draw rectangle
draw_rectangle(wnd, 10, 10, 100, 100, RGB(255, 0, 0));

// Handle events
while (1) {
    event_t *evt = wait_event(wnd);
    if (evt->type == EVENT_CLOSE) break;
}
```

---

## Summary Matrix

| Component | Purpose | Language | Architecture |
|-----------|---------|----------|--------------|
| Boot | Firmware bridge, kernel loading | Assembly | x86/ARM |
| Kernel | Core OS services | C++, Assembly | x86/ARM |
| XELoader | Dynamic loading | C++ | x86/ARM |
| Drivers | Hardware control | C++ | Generic |
| Libraries | APIs for apps | C++ | Generic |
| Applications | User programs | C/C++ | Generic |

This comprehensive guide provides the foundation for understanding every aspect of XenevaOS architecture and implementation.
