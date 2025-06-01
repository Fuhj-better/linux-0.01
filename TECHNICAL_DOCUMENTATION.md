# Linux 0.01 Internals: A Technical Overview

## 1. Introduction

This document provides a detailed exploration of the internal workings of the Linux kernel, version 0.01. Written by Linus Torvalds, this early version of Linux laid the groundwork for what would become a globally adopted operating system. It showcases a monolithic kernel architecture, implementing fundamental operating system concepts such as basic process management, a Minix v1 compatible filesystem, and essential hardware interactions for the Intel 80386 architecture.

This documentation will cover the following key areas:
-   **Boot Process**: How the system starts, from the BIOS to the execution of the first kernel code.
-   **Kernel Initialization**: The setup phase performed by `init/main.c`, including the transition to user mode.
-   **Kernel Core Systems**: Process management, interrupt handling, system calls, and basic device driver mechanisms.
-   **Memory Management**: Paging, copy-on-write, and physical memory organization.
-   **File System**: The Minix v1 filesystem implementation, buffer cache, inode, and file operations.
-   **Standard Library**: User-space C library wrappers for system calls and utility functions.
-   **Include Files**: Organization and content of critical header files.
-   **Build Process**: How the kernel source code is compiled and linked into a bootable image.

Understanding this early version offers valuable insights into the foundational design choices of the Linux kernel.

## 2. Boot Process (`boot/boot.s` and `boot/head.s`)

The journey of booting Linux 0.01 begins with the BIOS loading the initial boot sector and culminates in the kernel's 32-bit code preparing the system for multitasking. This process involves two key files: `boot/boot.s` and `boot/head.s`.

### 2.1. `boot/boot.s` - Initial Bootloader in Real Mode

The execution begins when the BIOS loads `boot.s` into memory at address `0x7C00`. This initial bootloader is responsible for preparing the system for the main kernel and transitioning from real mode to protected mode.

#### 2.1.1. Relocation and Initial Setup

-   **Self-Relocation**: The first action `boot.s` takes is to move itself from `0x7C00` to `0x90000`. This is done to free up the lower memory area (below 640KB) for the upcoming system code and BIOS data areas.
-   **Stack Setup**: A stack is set up at `0x90000 + 0x400` (i.e., `0x90400`).
-   **Displaying Message**: It uses BIOS interrupt `0x10` to print the "Loading system ..." message on the screen.

#### 2.1.2. Loading the System
-   The main operating system code (referred to as "system") is loaded from the boot device into memory at address `0x10000` (`SYSSEG`).
-   This is achieved by calling the `read_it` routine, which uses BIOS interrupt `0x13` to read sectors from the disk. It attempts to read whole tracks at a time for speed.
-   The `kill_motor` routine is called after loading to turn off the floppy drive motor.

#### 2.1.3. Moving the System to its Final Location
-   After the system is loaded to `0x10000`, interrupts are disabled using `cli`.
-   The loaded system code is then moved from `0x10000` down to `0x00000`. This is done in chunks to avoid overwriting itself prematurely.

#### 2.1.4. Preparing for Protected Mode
-   **Interrupt Controller Reprogramming**: The 8259 Programmable Interrupt Controllers (PICs) are reprogrammed.
    -   The BIOS typically maps hardware interrupts to `0x08-0x0F`. However, these conflict with Intel-reserved interrupt numbers.
    -   `boot.s` re-initializes the PICs to map hardware interrupts to vectors `0x20-0x2F`.
    -   All interrupts are masked off for now.
-   **A20 Gate Activation**: The A20 line is enabled. This is crucial for accessing memory above 1MB. It's done by sending commands (`0xD1` then `0xDF`) to the 8042 keyboard controller.
-   **Loading Descriptor Tables**:
    -   **IDT (Interrupt Descriptor Table)**: A dummy IDT is loaded with `lidt idt_48`. This IDT has a limit of 0, meaning no interrupts are defined yet.
    -   **GDT (Global Descriptor Table)**: A GDT is loaded using `lgdt gdt_48`. This GDT defines:
        -   A NULL descriptor.
        -   A code segment descriptor (8MB limit, base 0).
        -   A data segment descriptor (8MB limit, base 0).
        Both segments are configured for 386 protected mode with a 4KB granularity.

#### 2.1.5. Transition to Protected Mode
-   The PE (Protection Enable) bit in the `CR0` (Machine Status Word) register is set using the `lmsw ax` instruction (where `ax` holds `0x0001`).
-   Immediately after setting PE, a far jump `jmpi 0,8` is executed.
    -   This jumps to offset `0` within the code segment selected by descriptor index `1` in the GDT (selector `0x08`).
    -   This target address `0x00000000` is where `head.s` (specifically the `startup_32` label) begins execution.

### 2.2. `boot/head.s` - 32-bit Setup in Protected Mode

Execution transfers to `startup_32` in `head.s`, which is located at physical address `0x00000000`. This code runs in 32-bit protected mode and performs critical setup for the C kernel environment.

#### 2.2.1. Initial Setup (`startup_32`)
-   **Segment Registers**: Data segment registers (`ds`, `es`, `fs`, `gs`) are loaded with the selector for the data segment defined in the GDT (`0x10`, which is index 2).
-   **Stack**: The stack pointer (`esp`) is initialized using `lss _stack_start,%esp`.
-   **IDT Setup**:
    -   `call setup_idt` is executed.
    -   This routine initializes a proper IDT with 256 entries. Each entry is an interrupt gate pointing to a default handler `ignore_int`.
    -   The IDTR register is loaded with the address and limit of this new IDT (`lidt idt_descr`).
-   **GDT Setup**:
    -   `call setup_gdt` is executed.
    -   This routine loads a new GDT using `lgdt gdt_descr`. The new GDT is located at `_gdt` and contains:
        -   A NULL descriptor.
        -   Kernel Code Segment (CS): 8MB limit, base 0. Selector `0x08`.
        -   Kernel Data Segment (DS): 8MB limit, base 0. Selector `0x10`.
        -   Space for LDTs and TSSs.
    -   Segment registers (`ds`, `es`, `fs`, `gs`) are reloaded with `0x10` because the GDT has changed.
-   **A20 Line Check**: A quick check is performed to ensure the A20 line is indeed enabled.
-   **Math Coprocessor Check**:
    -   `CR0` is read.
    -   The ET (Extension Type) bit of `CR0` is checked. If set, a 387 math coprocessor is present.
    -   If not present, the EM (Emulate) bit in `CR0` is set to enable software emulation for floating-point instructions.
    -   `CR0` is updated.

#### 2.2.2. Paging Initialization
-   The code execution then jumps to `after_page_tables`.
-   **Parameter Pushing for `main`**: Before setting up paging, arguments for the `_main` C function (three zero values), its return address (`L6`), and the address of `_main` itself are pushed onto the stack.
-   **`jmp setup_paging`**: Control is transferred to the `setup_paging` routine.
    -   **Page Directory and Page Tables**:
        -   The page directory is initialized at physical address `0x00000000`. This means the `startup_32` code and earlier setup routines are overwritten.
        -   The first two entries of the page directory are set to point to `pg0` and `pg1` (page tables).
        -   `pg0` and `pg1` are initialized to identity-map the first 8MB of physical memory.
        -   Each page table entry is set with Present and Read/Write flags.
    -   **CR3 Load**: The `CR3` register (Page Directory Base Register - PDBR) is loaded with the physical address of the page directory (`0x00000000`).
    -   **Enable Paging**: The PG bit (bit 31) in `CR0` is set, enabling paging.
    -   **Return**: `setup_paging` returns. This `ret` instruction also flushes the CPU's prefetch queue.

#### 2.2.3. Jumping to `main()`
-   When `setup_paging` executes its `ret` instruction, it pops the return address from the stack.
-   Because `_main`'s address was pushed before `setup_paging` was jumped to, `setup_paging` effectively returns to `_main`.
-   The `_main` C function (the kernel entry point in `init/main.c`) begins execution.
-   The label `L6: jmp L6` serves as a trap in case `_main` ever attempts to return.

This concludes the initial hardware setup and transition to a 32-bit protected mode environment with paging enabled, ready for the C kernel to take over.

## 3. Kernel Initialization (`init/main.c`)

The `main()` function in `init/main.c` is the first C code to run. It initializes core kernel subsystems and transitions to user mode by starting the first processes. Interrupts are initially disabled.

### 3.1. Core Kernel Subsystem Initializations
- **`time_init()`**: Reads time from CMOS, converts from BCD, stores in `startup_time`.
- **`tty_init()`**: Initializes the TTY subsystem for console I/O.
- **`trap_init()`**: Sets up IDT gates for CPU exceptions and hardware interrupts (actual handlers are in `kernel/traps.c` and `asm.s`).
- **`sched_init()`**: Initializes the scheduler, Task 0, and the PIT for timer interrupts.
- **`buffer_init()`**: Initializes the buffer cache for block device I/O.
- **`hd_init()`**: Initializes the hard disk controller and driver.
- **`sti()`**: Enables maskable hardware interrupts.

### 3.2. Transition to User Mode and Process Creation
- **`move_to_user_mode()`**: Switches CPU from kernel (Ring 0) to user mode (Ring 3) for Task 0.
- **Forking Process 1**:
    - Task 0 calls `fork()` (an inline system call) to create Process 1.
    - **Process 1 (child)**: `fork()` returns 0. Calls `init()`.
    - **Task 0 (parent)**: `fork()` returns Process 1's PID. Task 0 then enters an idle loop: `for(;;) pause();`. `pause()` for Task 0 yields the CPU, making it the idle task.
- **`init()` Function (Process 1)**:
    - `setup()`: System call, likely mounts the root filesystem (details depend on `ROOT_DEV`).
    - Forks a child (Process 2) to execute `/bin/update` (typically for filesystem sync). Process 1 does not wait for it.
    - Sets up standard I/O (stdin, stdout, stderr) for itself to `/dev/tty0`.
    - Forks another child (Process 3).
        - **Process 3**: Calls `setsid()` to create a new session, reopens `/dev/tty0`, then attempts `execve("/bin/sh", ...)` to start the system shell. This shell is the primary user-space process.
    - Process 1 calls `wait()` for a child to terminate (typically Process 3 if the shell exits).
    - Calls `sync()` and then `_exit(0)`.

If `/bin/sh` (Process 3) exits, Process 1 also exits. In this early kernel, this would leave Task 0 idling and any detached daemons running.

## 4. Kernel Core Systems (`kernel/`)

This section details core kernel functionalities: process management, interrupt/exception handling, system calls, and basic device drivers.

### 4.1. Process Management
Covers process creation, scheduling, and termination.

#### 4.1.1. Process Creation (`kernel/fork.c`)
- **`sys_fork()`**: System call entry.
- **`find_empty_process()`**: Finds an available slot in the `task` array and assigns a new PID.
- **`copy_process()`**: Core fork logic:
    - Allocates a page for the new `task_struct` and kernel stack.
    - Copies parent's `task_struct`.
    - Initializes PID, parent PID, counter, signal handlers, times.
    - Sets up the new process's TSS (Task State Segment), including `esp0` (kernel stack pointer), `ss0`, and copies parent's registers. Saves FPU state if necessary.
    - **`copy_mem()`**: Duplicates memory space using `copy_page_tables()`, which implements Copy-on-Write (COW) by marking pages read-only for both parent and child.
    - Increments file and inode reference counts for shared resources.
    - Sets up GDT entries for the new task's TSS and LDT.

#### 4.1.2. Scheduler (`kernel/sched.c`)
- **`schedule()`**:
    - Handles alarms and wakes up interruptible tasks with pending signals.
    - Selects the next task to run from `TASK_RUNNING` processes based on the highest `counter` (time slice).
    - If no task has `counter > 0`, recalculates counters for all tasks: `counter = (counter >> 1) + priority`.
    - `switch_to(next)`: Performs context switch (details in `asm/system.h`).
- **Task States**: `TASK_RUNNING`, `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `TASK_ZOMBIE`.
- **Synchronization Primitives**:
    - `sleep_on(struct task_struct **p)`: Puts current task to `TASK_UNINTERRUPTIBLE` sleep on wait queue `p`.
    - `interruptible_sleep_on(struct task_struct **p)`: Similar, but `TASK_INTERRUPTIBLE`.
    - `wake_up(struct task_struct **p)`: Wakes up a task sleeping on `p`.
- **`do_timer()`**: Called by timer interrupt; updates process times (`utime`, `stime`), decrements `counter`, and calls `schedule()` if counter is zero and in user mode.
- **`sched_init()`**: Initializes Task 0, PIT for `HZ` frequency, timer interrupt gate (0x20), and system call gate (0x80).

#### 4.1.3. Process Termination (`kernel/exit.c`)
- **`sys_exit()` / `do_exit()`**:
    - Frees page tables.
    - Reparents children to process 0 (implicitly, by setting `father = 0`).
    - Closes open files. Releases PWD and root inodes.
    - If it has a parent, becomes `TASK_ZOMBIE`, stores `exit_code`, and sends `SIGCHLD` to parent (`do_kill()`).
    - If orphaned, calls `release()` directly.
    - Calls `schedule()`.
- **`release()`**: Frees the task structure page.
- **`sys_waitpid()`**: Waits for child process state changes, collects zombie exit status and resources.
- **`sys_kill()` / `do_kill()` / `send_sig()`**: Implements signal sending logic.

#### 4.1.4. Other Process System Calls (`kernel/sys.c`)
- Getters: `sys_getpid()`, `sys_getppid()`, `sys_getuid()`, `sys_geteuid()`, `sys_getgid()`, `sys_getegid()`, `sys_getpgrp()`.
- Setters: `sys_setuid()`, `sys_setgid()`, `sys_setpgid()`, `sys_setsid()`, `sys_nice()`, `sys_umask()`.
- Memory: `sys_brk()`.
- Signals: `sys_signal()`, `sys_alarm()`, `sys_pause()`.
- Time: `sys_time()`, `sys_stime()`, `sys_times()`.
- Info: `sys_uname()`.
- Many others are stubs (`-ENOSYS`).

### 4.2. Interrupts, Exceptions, and System Calls

#### 4.2.1. Trap Handling (`kernel/traps.c`)
- **`trap_init()`**: Sets up IDT gates for CPU exceptions (0-16) using `set_trap_gate` (interrupts disabled on entry) and `set_system_gate` (callable from user mode, DPL=3 for int3, overflow, bounds).
- **Exception Handlers**: Functions like `do_divide_error()`, `do_general_protection()`, etc., typically call `die()` to print debug info and terminate the process (`do_exit(SIGSEGV)`).
- **Page Faults**: Interrupt 14, handled by `page_exception` (in `mm/page.s`).

#### 4.2.2. System Call Mechanism (`kernel/system_call.s`)
- **`_system_call` (Interrupt 0x80)**:
    - Kernel entry point for system calls.
    - Saves registers, switches to kernel stack.
    - Calls C handler via `_sys_call_table[syscall_nr]`.
    - After C handler returns, checks `current->state` and `current->counter` for rescheduling.
- **`ret_from_sys_call`**:
    - Handles pending signals before returning to user mode. Sets up user stack for signal handler if needed.
    - Restores registers and uses `iret` to return.
- **`_timer_interrupt`**: IRQ0 handler. Updates `jiffies`, calls `do_timer()`, then jumps to `ret_from_sys_call`.
- **`_hd_interrupt`**: IRQ14 handler. Calls function pointed by `do_hd`, then `iret`.

### 4.3. Device Drivers (Overview)
Brief descriptions of key device driver interactions.

#### 4.3.1. Hard Disk (`kernel/hd.c`)
- Manages block requests using an elevator algorithm in `request[]` queue.
- `hd_init()`: Sets up IRQ14.
- `rw_hd()` / `rw_abs_hd()`: Queue block read/write requests.
- `do_request()`: Issues commands to the HD controller.
- `read_intr()` / `write_intr()`: Interrupt handlers for data transfer.
- `sys_setup()`: Reads partition table.

#### 4.3.2. Console and Keyboard (`kernel/console.c`, `kernel/keyboard.s`)
- **`console.c`**: VT102 emulation, direct video memory access, escape sequence parsing. `con_init()` sets up IRQ1. `con_write()` outputs to screen.
- **`keyboard.s`**: `_keyboard_interrupt` (IRQ1) reads scan codes, uses `key_table` to call handlers (`do_self`, `cursor`, `func`, etc.), and puts characters into TTY queue via `put_queue`.

#### 4.3.3. Serial Port (`kernel/serial.c`)
- `rs_init()`: Configures COM1 (IRQ4) and COM2 (IRQ3), sets baud rate (2400bps).
- `rs_write()`: Enables transmit interrupt when data is available.
- Interrupt handlers (in assembly) manage data reception/transmission.

### 4.4. Time Management (`kernel/mktime.c`)
- **`kernel_mktime()`**: Converts `struct tm` to seconds since epoch.
- **`jiffies`**: Global tick counter, incremented by timer interrupt (`HZ` times/sec).
- **`startup_time`**: System boot time in seconds. `CURRENT_TIME` provides current Unix time.

### 4.5. Logging and Output (`kernel/printk.c`, `kernel/vsprintf.c`)
- **`printk()`**: Kernel's `printf`, uses `vsprintf()` to format string into a buffer, then `tty_write()` for console output.
- **`vsprintf()`**: Standard string formatting utility.

## 5. Memory Management (`mm/`)

Linux 0.01 uses paging with a two-level page table structure (Page Directory, Page Table), supporting demand paging and Copy-on-Write (COW).

### 5.1. Paging Architecture Overview
- **Two-Level Paging**: Page Directory (`_pg_dir` at 0x0) and Page Tables. Each PT maps 4MB.
- **Linear Address Space**: Each process has a 64MB linear address space (`task_nr * 0x4000000`).
- **TLB Invalidation**: `invalidate()` macro reloads `CR3`.

### 5.2. Core Memory Management (`mm/memory.c`)
- **Physical Memory**: `LOW_MEM` (start of dynamic allocation, ~1MB), `HIGH_MEMORY`. `mem_map[]` array tracks page usage (reference counts).
- **Page Allocation/Deallocation**:
    - `get_free_page()`: Finds a free page in `mem_map`, marks used, zeroes, returns address.
    - `free_page()`: Decrements page count in `mem_map`; if 0, marks free.
- **Page Table Manipulation**:
    - `put_page()`: Maps a physical page to a linear address, allocating PT if needed.
    - `copy_page_tables()`: For `fork()`. Copies page mappings, marks pages read-only for COW, increments `mem_map` counts.
    - `free_page_tables()`: For `exit()`. Frees pages and page tables.
- **Page Fault Handling**:
    - **`do_no_page()` (Not Present Fault)**: Allocates a new page (`get_free_page()`) and maps it (`put_page()`). No disk loading/swapping.
    - **`do_wp_page()` (Write Protect Fault / COW)**: Calls `un_wp_page()`.
    - **`un_wp_page()`**: If page count is 1, makes writable. Else, allocates new page, copies content, updates PTE, decrements old page count.
    - **`write_verify()`**: Proactive COW for `verify_area`.
- **OOM**: `do_exit(SIGSEGV)` if `get_free_page()` fails.
- **`sys_brk`**: Updates `current->brk`; allocation is demand-paged.

### 5.3. Low-Level Page Fault Handling (`mm/page.s`)
- **`_page_fault` (Interrupt 14)**: Assembly entry. Saves state, gets faulting address (`CR2`) and error code. Calls `_do_no_page` (if P=0) or `_do_wp_page` (if P=1 and write fault). Restores state and `iret`.

## 6. File System (`fs/`)

Linux 0.01 uses a Minix v1 compatible filesystem, with a buffer cache for disk I/O.

### 6.1. Filesystem Architecture
- Minix v1: 1KB blocks, inodes, directories as files, bitmaps for free blocks/inodes.
- Buffer Cache (`fs/buffer.c`): System-wide cache for disk blocks (`struct buffer_head`).
- In-memory inodes (`inode_table` in `fs/inode.c`).
- Open file structures (`file_table` in `fs/file_dev.c`).

### 6.2. Buffer Cache (`fs/buffer.c`)
- `struct buffer_head` (`bh`): Contains data pointer, dev/blocknr, flags (`b_uptodate`, `b_dirt`), count, lock.
- Hash table (`hash_table`) for quick lookup; `free_list` for LRU-like management.
- **`getblk(dev, block)`**: Gets a buffer, potentially reusing from free list or waiting if none available. Handles dirty buffers.
- **`bread(dev, block)`**: `getblk()` then `ll_rw_block(READ, bh)` if not up-to-date.
- **`brelse(buf)`**: Releases a buffer (decrements count).
- **`sys_sync()`**: Writes all dirty inodes and buffers to disk.

### 6.3. Bitmap Management (`fs/bitmap.c`)
- `free_block()`, `new_block()`: Manipulate zone (block) bitmaps.
- `free_inode()`, `new_inode()`: Manipulate inode bitmaps.
- Use `set_bit()`, `clear_bit()`, `find_first_zero()` on bitmap blocks.

### 6.4. Inode Management (`fs/inode.c`)
- `struct m_inode` (in-memory) vs `struct d_inode` (on-disk).
- `inode_table[]`: Cache of active inodes.
- **`iget(dev, nr)`**: Retrieves an inode from cache or reads from disk (`read_inode()`). Increments `i_count`.
- **`iput(inode)`**: Releases an inode. Decrements `i_count`. If 0 and `i_nlinks` is 0, truncates and frees. If dirty, writes back (`write_inode()`).
- `lock_inode()`, `unlock_inode()`: Simple inode locking.
- **`bmap(inode, block)` / `create_block(inode, block)`**: Map logical file blocks to physical disk blocks, handling direct/indirect/double-indirect zones.

### 6.5. Pathname Resolution (`fs/namei.c`)
- **`namei(pathname)`**: Resolves full path to an inode.
- **`dir_namei(pathname, ...)`**: Resolves parent directory of a path.
- **`get_dir(pathname)`**: Traverses path components.
- **`find_entry(dir, name, ...)`**: Searches directory for an entry.
- **`add_entry(dir, name, ...)`**: Adds entry to a directory.
- `permission()`: Checks R/W/X permissions.
- Implements `sys_mkdir`, `sys_rmdir`, `sys_link`, `sys_unlink`.

### 6.6. File Operations (`fs/file_dev.c`, `fs/read_write.c`)
- `file_table[]`: Global open file array. `current->filp[]` points here.
- **`sys_read()` / `sys_write()`**: Dispatch based on inode type (pipe, char, block, regular/dir).
- **`file_read()` / `file_write()`**: For regular files. Use `bmap`/`create_block` and `bread` to interact with buffer cache.
- **`sys_lseek()`**: Modifies `file->f_pos`.

### 6.7. File Opening/Creation (`fs/open.c`)
- **`sys_open(filename, flag, mode)`**: Allocates `fd` and `struct file`. Calls `open_namei()` (which can create file via `new_inode` and `add_entry` if `O_CREAT`). Initializes `struct file`.
- **`sys_creat()`**: Calls `sys_open()` with `O_CREAT | O_TRUNC`.
- Other syscalls: `sys_utime`, `sys_access`, `sys_chdir`, `sys_chroot`, `sys_chmod`, `sys_chown`.

### 6.8. Superblock Operations (`fs/super.c`)
- `struct super_block`: In-memory superblock. Holds FS metadata and bitmap buffer pointers.
- **`mount_root()`**: Called at boot. `do_mount(ROOT_DEV)` reads superblock and bitmaps. Gets root inode.
- No dynamic mount/umount.

### 6.9. Device Abstraction (`fs/char_dev.c`, `fs/block_dev.c`)
- **`char_dev.c`**: `crw_table[]` (array of function pointers) for char device R/W, indexed by MAJOR. `rw_char()` dispatches.
- **`block_dev.c`**: `rd_blk[]` for block device requests. `ll_rw_block()` dispatches to device-specific request function (e.g., `rw_hd`). `block_read()`/`block_write()` are generic helpers using `bread`.

### 6.10. Pipe Implementation (`fs/pipe.c`)
- **`sys_pipe(fildes)`**: Creates a pipe. Gets an inode via `get_pipe_inode()` (which allocates a page for the buffer). Sets up two `struct file` entries (one read, one write) pointing to this inode.
- `read_pipe()`, `write_pipe()`: Implement blocking read/write on the circular page buffer, using `sleep_on`/`wake_up` on `inode->i_wait`.

### 6.11. Other Utilities
- **`fs/fcntl.c`**: `sys_dup`, `sys_dup2`, `sys_fcntl` (F_DUPFD, F_GETFD, F_SETFD, F_GETFL, F_SETFL).
- **`fs/ioctl.c`**: `sys_ioctl` dispatching via `ioctl_table[]`.
- **`fs/stat.c`**: `sys_stat`, `sys_fstat`.
- **`fs/truncate.c`**: `truncate()` frees all blocks of an inode.
- **`fs/exec.c`**: `namei()` to find executable, `bread()` for header, `read_area()` to load text/data.

## 7. Standard Library (`lib/`)

The C library in `lib/` provides user-space programs with interfaces to kernel system calls and common utilities. Many are thin wrappers using `_syscallN` macros or direct `int $0x80` for system calls.

- **`_exit.c` (`_exit()`)**: Wrapper for `__NR_exit` system call.
- **`close.c` (`close()`)**: Wrapper for `__NR_close` via `_syscall1`.
- **`ctype.c`**: Defines `_ctype` array for character classification macros.
- **`dup.c` (`dup()`)**: Wrapper for `__NR_dup` via `_syscall1`.
- **`errno.c`**: Defines global `errno`.
- **`execve.c` (`execve()`)**: Wrapper for `__NR_execve` via `_syscall3`.
- **`open.c` (`open()`)**: Wrapper for `__NR_open` using direct assembly for variadic arguments.
- **`setsid.c` (`setsid()`)**: Wrapper for `__NR_setsid` via `_syscall0`.
- **`string.c`**: Includes `<string.h>`; actual implementations often GCC builtins.
- **`wait.c` (`wait()`, `waitpid()`)**: `waitpid()` uses `_syscall3`; `wait()` calls `waitpid()`.
- **`write.c` (`write()`)**: Wrapper for `__NR_write` via `_syscall3`.
- **`_syscallN` Macros**: Defined in `include/unistd.h`, these generate inline assembly for system calls, handling register setup and `errno`.

## 8. Include Files (`include/`)

Header files in `include/` and its subdirectories (`asm/`, `linux/`, `sys/`) define structures, constants, macros, and function prototypes for kernel and user-space.

### 8.1. Overall Organization
- **`include/`**: Standard C library style and POSIX-like headers.
- **`include/asm/`**: Architecture-specific (i386) definitions, inline assembly for I/O, segment access, system control.
- **`include/linux/`**: Linux kernel-specific definitions (scheduler, filesystem, MM, TTY structures).
- **`include/sys/`**: System-level types and POSIX-like structures.

### 8.2. Key General Headers (in `include/`)
- **`const.h`**: Inode type/permission constants.
- **`ctype.h`**: Character classification macros.
- **`errno.h`**: Error number definitions.
- **`fcntl.h`**: `open()` flags, `fcntl()` commands.
- **`signal.h`**: Signal numbers, `struct sigaction`.
- **`stdarg.h`**: Variadic function macros.
- **`stddef.h`**: `size_t`, `NULL`, `offsetof`.
- **`string.h`**: String/memory function declarations.
- **`termios.h`**: Terminal I/O structures and constants.
- **`time.h`**: Time types (`time_t`) and `struct tm`.
- **`unistd.h`**: `__NR_xxx` syscall numbers, `_syscallN` macros, POSIX constants.
- **`utime.h`**: `struct utimbuf`.

### 8.3. `include/asm/` Specific Headers
- **`io.h`**: `inb()`, `outb()`, `inb_p()`, `outb_p()` for port I/O.
- **`memory.h`**: Inline `memcpy`.
- **`segment.h`**: `get_fs_byte()`, `put_fs_long()` etc., for `fs` segment access.
- **`system.h`**: `move_to_user_mode()`, `sti()`, `cli()`, GDT/IDT setup macros.

### 8.4. `include/linux/` Specific Headers
- **`config.h`**: Kernel compile-time options (`HIGH_MEMORY`, `ROOT_DEV`).
- **`fs.h`**: Filesystem structures (`buffer_head`, `m_inode`, `file`, `super_block`).
- **`hdreg.h`**: Hard disk controller registers/commands.
- **`head.h`**: `struct desc_struct`, `pg_dir`, `idt`, `gdt` declarations.
- **`kernel.h`**: `panic()`, `printk()` declarations.
- **`mm.h`**: `PAGE_SIZE`, memory function prototypes.
- **`sched.h`**: `struct task_struct`, `struct tss_struct`, task states, `INIT_TASK`.
- **`sys.h`**: `sys_call_table[]` definition.
- **`tty.h`**: `struct tty_struct`, `struct tty_queue`.

### 8.5. `include/sys/` Specific Headers
- **`stat.h`**: `struct stat`, file mode macros.
- **`times.h`**: `struct tms`.
- **`types.h`**: Basic system types (`pid_t`, `dev_t`, `ino_t`, `off_t`).
- **`utsname.h`**: `struct utsname`.
- **`wait.h`**: `waitpid()` options and status macros.

### 8.6. Other Notable Headers
- **`a.out.h`**: `struct exec` for a.out executable format.

## 9. Build Process (`Makefile` and `tools/build.c`)

The Linux 0.01 kernel is built using a `Makefile` and the `tools/build` utility, which assemble the boot sector and kernel code into a bootable `Image`.

### 9.1. `Makefile`
- **Targets**: `all`, `Image`, `tools/system`, `boot/boot`.
- **Compilation Flags**: `CFLAGS` (for GCC), `AS86FLAGS` (for 16-bit `as`), `LD86FLAGS`.
- **Sequence**:
    1.  Builds kernel components (`kernel/*.o`, `mm/*.o`, `fs/*.o`), libraries (`lib/lib.a`), and `init/main.o` via recursive makes.
    2.  Compiles `boot/head.s` (32-bit assembly start).
    3.  Links `boot/head.o`, `init/main.o`, kernel archives, and `lib.a` into `tools/system`. `System.map` is generated.
    4.  Calculates `SYSSIZE` from `tools/system` size, prepends to `boot/boot.s`, then assembles and links `boot/boot` (16-bit bootloader).
    5.  Compiles `tools/build.c` into `tools/build`.
- **Image Construction**: `tools/build boot/boot tools/system > Image` concatenates the boot sector and the main kernel.

### 9.2. `tools/build.c`
- **Purpose**: Combines `boot/boot` (boot sector) and `tools/system` (kernel) into the final `Image`.
- **Operation**:
    - Reads `boot/boot`, validates its Minix a.out header.
    - Ensures boot sector code is <= 510 bytes, adds 0x55AA signature. Writes 512 bytes to stdout.
    - Reads `tools/system`, validates its a.out header (entry point 0).
    - Appends entire content of `tools/system` to stdout.
- The `Image` is thus `boot/boot` (512 bytes) followed by `tools/system`.

## 10. Conclusion

This document has provided a technical walkthrough of the Linux 0.01 kernel, covering its boot sequence, initialization, core systems (process management, memory management, filesystem), standard library, include file structure, and build process. This early version, while simple by today's standards, established many of the fundamental architectural principles that guided Linux's future development.
