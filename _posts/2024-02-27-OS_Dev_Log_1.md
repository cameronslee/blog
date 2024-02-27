---
title: OS Development Log 1
date: 2024-02-27
---

I have been working on an operating system for 2 weeks now and I am here to
report what I have learned.

I started from the ground up. After wheeling through and reading parts of the 
OS Dev Wiki, I found myself with a canvas including space for a kernel to 
grow, I was off to the races of building my first operating system: Chimp OS.

```
/* ============ Chimp OS: a primitive operating system ========== */

        Specs:
        - x86 32 bit operating system with paging (TBD)
        
        Requirements:
        - i686-elf-gcc compiler (GCC)
        - Linker and Assembler (Binutils)
        - QEMU
        - Bootloader (GRUB)

/* ============================================================== */
```
## First step, a cross-compiler
The first thing I did when starting this project was set up a cross-compiler to 
allow me to compile my new OS from the existing operating system on my machine.

This consisted of obtaining and building:
- GNU Binutils which provides the GNU Assembler and Linker.
- GCC to provide the necessary compiler collection.

As I am targeting x86, I built the i686-elf-gcc cross-compiler.

## Bootloaders
The bootloader is responsible for:
    - Bringing the kernel into memory
    - Providing an environment for the kernel to operate
    - Transfer control to the kernel.

### Real Mode
Turns out in x86, the bootloader runs in real mode. Real mode is a 16 bit mode
that allows for easy access to BIOS resources and routines.

I used GRUB as my bootlader. Perhaps one day I'll roll my own bootloader but I 
wanted to get straight into working on a kernel.

This came with creating some bootstrap assembly to set up the multiboot header, 
the stack and a main entry point for the kernel

### Multiboot - Defines how a bootloader and OS interface
Configuring multiboot is the first thing that comes. The multiboot standard 
defines the interface for which an operating system and the boot loader 
interact.
```
# Declare constants for the multiboot header.
.set ALIGN,    1<<0             # align loaded modules on page boundaries
.set MEMINFO,  1<<1             # provide memory map
.set FLAGS,    ALIGN | MEMINFO  # this is the Multiboot 'flag' field
.set MAGIC,    0x1BADB002       # 'magic number' lets bootloader find header
.set CHECKSUM, -(MAGIC + FLAGS) # checksum of above, to prove we are multiboot

# Declare a multiboot header that marks the program as a kernel.
.section .multiboot.data, "aw"
.align 4
.long MAGIC
.long FLAGS
.long CHECKSUM

```

### Setting up a stack
One thing to note with the x86 architecture is that the stack grows downwards 
in memory.

```
                    ----------------
                    | Stack Bottom |
                    |              |
                    ----------------
                    ----------------
          -->       |              |  We reserve 16KiB of memory by 'skipping'
          |         |              |  16384 bits from the stack bottom and 
          |         ----------------  placing the stack top in its place.
          |         ----------------
    16KiB |         |              |
          |         |              |
          |         ----------------
          |         ----------------
          |         | Stack Top    |
          -->       |              |
                    ----------------

```

Here is the code

```
# Allocate the initial stack.
.section .bootstrap_stack, "aw", @nobits
stack_bottom:
.skip 16384 # 16 KiB
stack_top:

```

#### Paging
I am still on the fence about paging in this OS. Paging provides virtual memory
and allows us to have an execution space that is unbounded by the systems 
physical memory. However it brings another layer of complexity when accessing 
frames of memory. 

As I see it, an operating system with paging will allow for greater depth and 
usability but one without will be easier to debug. I decided to go ahead and 
enable paging for now. When I cross into the development of a user space I 
will reassess the need for paging.

At a high level, the MMU (Memory Management Unit) maps address
spaces through the following two tables:

##### Paging Directory
- Each entry here points to the page table
##### Paging Table 
- Each entry points to a 4 KiB physical page frame.
- each entry has bits controlling access protection and caching features.

Both of these tables contain 1024 4 byte entries (4 KiB each). The entire 
paging system represnts a 4GB virtual memory map!

Translation of virtual addresses to physical involves dividing the virtual 
address into three parts:
- bits 22-31, which specify the index of the page directory
entry
- bits 12-21 which specifcy the index of the page table
- bits 0-11 which specify the page offset

The MMU will use the paging structures by starting with the page directory,
use the entry to access the page table, use the page table entry to access 
the base address of the physical page frame and then apply the offset to the
physical base address to produce the physical address.

```
Page Directory Entry
--------------------------------------------------------------------
|31          22  21  20          13  12  11    9  8 7 6 5 4 3 2 1 0| 
|                                                                  |
| Index              Page table             Offset                 |
|                    index                                         |
--------------------------------------------------------------------

Page Table Entry
--------------------------------------------------------------------
|31                                  12  11    9  8 7 6 5 4 3 2 1 0| 
|                                                                  |
|Physical address base                                             |
|                                                                  |
--------------------------------------------------------------------

// These diagrams are simplified and do not contain all of the additonal
// Fields present in the lower bits!

```
If translation of a virtual address to a physical address fails, the processor
will issue a **page fault**

### Kernel Entry Point
This is the point where we define the start of the kernel. In the bootstrapped
assembly contains a call to the main implementation of the kernel in C where
VGA buffers are set up for drawing to the screen as well as any other necessary
things to initialize. From here control is passed off to the kernel

## The Kernel 
On to the kernel for real now. 

### Global Descriptor Table (GDT)
This is a data structure for the x86 architecture. The entries in this table 
contain information for the CPU regarding memory segments.

#### Segmentation
Segments of memory refer to a contiguous chunk of physical memory.

In real mode, memory segments are referenced by:
- address = A:B where A is the base physical address and B is the offset.

In protected mode, we still use a base address and physical address. However, 
this information lives in our GDT.


Back to the GDT. Here is a brief visualization of each entry.
```
/*                        Segment Descriptor 
 * --------------------------------------------------------------
 * |63        56  55    52  51    48  47        40  39        32|                 
 * --------------------------------------------------------------
 * |Base          Flags     Limit     Access        Base        |
 * |31        24  3     0   19    16  7         0   23        16|
 * --------------------------------------------------------------
 * |31                            16  15                       0|
 * --------------------------------------------------------------
 * |Base                              Limit                     |
 * |15                            0   15                       0|
 * --------------------------------------------------------------
 *
*/
```

Segmentation in this manner provides memory protection as well as different 
regions such as code, data, stack etc.

As for the implementation, I went ahead and implemented the segment descriptor 
above and set up functions to initialize it. 

The GDT must contain the following three entries to start:
- Null Descriptor
- Code Segment Descriptor
- Data Segment Descriptor

Here are the steps to set up the GDT:
- Disable Interrupts (CLI asm instruction)
- Fill the the table
- Tell the CPU where to look for the GDT (load LGDT register with GDT pointer!)

Once this is set up we can enter protected mode (crucial!)

#### Protected Mode
With the GDT and paging setup, we have access to the full physical and virtual
address space. We will stay here even when we enter user land.

### Interrupt Descriptor Table (IDT)
The IDT is used to define how we handle interrupts. For intel, 32 spaces are
reserved for set routines on how to handle exceptions. We define a data
structure to hold this information as well as a fault handler to be called 
when an exception is reached.

```
/*       IDT Entry
 * -----------------------
 * |7  6    5  4        0|
 * -----------------------
 * |P  DPL     always    |
 * -----------------------
 *
 * P: segment present? 1 = yes
 * DPL: Ring (0-3)
 * Always: 01110 (lower bits set to 01110)
*/
```
We populate the table with the following 32 interrupt service routines (ISR):
```
1. Divide By Zero Exception
2. Debug Exception
3. Non Maskable Interrupt Exception
4. Int 3 Exception
5. INTO Exception
6. Out of Bounds Exception
7. Invalid Opcode Exception
8. Coprocessor Not Available Exception
9. Double Fault Exception (With Error Code)
10. Coprocessor Segment Overrun Exception
11. Bad TSS Exception (With Error Code)
12. Segment Not Present Exception (With Error Code)
13. Stack Fault Exception (With Error Code)
14. General Protection Fault Exception (With Error Code)
15. Page Fault Exception (With Error Code)
16. Reserved Exception
17. Floating Point Exception
18. Alignment Check Exception
19. Machine Check Exception
20. Reserved
21. Reserved
22. Reserved
23. Reserved
24. Reserved
25. Reserved
26. Reserved
27. Reserved
28. Reserved
29. Reserved
30. Reserved
31. Reserved
32. Reserved

```
Just like the GDT, we load the pointer to the table we defined into the LIDT
register. With that we have ISRs set up.

#### Custom Interrupt Requests
Now it is possible for us to trigger our own software or hardware exceptions.
This is similar to setting up the IDT. We do this by 

### Programmable Interval Timer (PIT) 
Also called a system clock, is used to generate interrupts at regular time 
intervals.

The chip itself has the following channels:
- Channel 0: tied to IRQ0 to interupt the CPU
- Channel 1: system specific channel
- Channel 2: System speaker

```
/*
 * Programmable Interval Timer
 * -----------------------
 * |7   6 5  4 3     1  0|
 * -----------------------
 * |CNTR  RW   Mode  BCD |
 * -----------------------
 * CNTR = Counter num (0-2)
 * RW =  Read Write Mode
 * Mode =
 *   0     Interrupt on terminal count
 *   1     Hardware retriggerable one shot
 *   2     Rate generator
 *   3     square wave  
 *   4     software strobe 
 *   5     hardware strobe 
 *
 * BCD = binary coded decimal
 *   0 = 16 bit counter
 *   1 = 4x BCD decay counter
 */
```

With this, I implemented a timer_wait() that waits for a set amount of time
before continuing.

After this, I implemented a system uptime counter. This will be used to
manage time based processes in the near future.

### Keyboard Input 
Finally it has come time to accept keyboard input which will segway into 
the development of a user space. In order to do this, a custom interrupt 
request handler has to be made.

As custom interrupt requests are already implemented, I used IRQ1 to handle
the keyboards input.

Now whenever a key is pressed, the keyboard data buffer is read for a scan
code and issues a lookup to determine the key and character that was pressed.

```
void keyboard_handler(struct regs *r)
{
    unsigned char scancode;

    /* Read from kb data buffer */
    scancode = inportb(0x60);

    /* If the top bit of the byte we read from kb is
     *  set, that means that a key has just been released 
    */
    if (scancode & 0x80)
    {
        // Handle shift, alt, ctrl
    }
    else
    {
        /* Handle key presses. If held , multiple interrupts will be issued */
    }
}
```

With this synced up with the system uptime counter, keyboard input is supported 

# Conclusion
I have learned alot so far. While this is merely setting the groundwork for
greater things, I am excited to continue iterating on this. 

OS Dev wiki and the old archives of osdever have been an incredible resource.

User space and memory management is next.

Recreational OS Development is something special.
