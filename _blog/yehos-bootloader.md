---
title: Building a Bootloader
date: 2017-06-17T10:20:00Z
keywords: Operating Systems
---

When you power on your computer, firmware on your computer's system board loads a single sector of your boot drive into memory. The program, called the bootloader, is responsible for loading the kernel, preparing the processor to operate in the right mode, and beginning execution. In this post I'll walk through process of booting into protected mode on x86 hardware by describing the boot sequence for Yehos, a simple operating system for the X386 processor.

[Yehos](https://github.com/zormit/yehos) was created by Andrea Law, Dominic Spadacene, Moritz Neeb, Nandaja Varma, Saul Pwanson, and Vivian Brown to teach the fundamentals of operating systems development. Our annotated bootloader.asm is [available on Github](https://github.com/zormit/yehos/blob/master/bootloader.asm).

## The boot sequence

### 1. Press power to start the BIOS

When a user presses the power button, the computer hardware fires a reset signal that starts the BIOS (Basic Input/Output System). The BIOS is initialization code and a set of basic routines, embedded on the motherboard in ROM (Read-Only Memory).

The BIOS does some basic hardware checks to make sure the keyboard, monitor, and other devices are working.

### 2. The BIOS loads the bootloader

Next, the BIOS looks for a bootloader. It searches each storage device for a "boot signature", a special sequence of bits that indicates the presence of a bootloader. Ours is appended to the end of our bootloader like this:

{% highlight nasm %}
times (512 - $ + entry - 2) db 0 ; pad boot sector with zeroes
db 0x55, 0xAA                    ; 2 byte boot signature
{% endhighlight %}

When the BIOS finds the boot signature, it loads the preceding block into main memory and starts executing at the beginning of that block. At this point, we're in our bootloader.

### 3. Prepare for boot

We start of with a bit of housekeeping by:

1. Turning off interrupts. We don't want a hardware interrupt to get in the way of the boot process.
2. Zeroing out all registers.
3. Setting up the boot stack by moving a known address into the stack pointer.

### 4. Get the address of the kernel

The BIOS only loaded a single sector of our kernel into memory (the bootloader). We need to load the rest of the kernel. But first, we need to tell the bootloader where to find it.

Because we're storing our kernel on an ISO image (a disk image that includes a filesystem), we don't know before the build exactly where it will be located. To help find it later, at build time we write the address of the kernel to the first sector of the ISO.

We've populated the beginning of our bootloader with a Disk Address Packet (DAP) that contains information about loading data off the ISO into memory. Our DAP looks like this:

{% highlight nasm %}
iso_boot_info:
bi_pvd  dd 16           ; LBA (logical block address) of primary volume descriptor
bi_file dd 0            ; LBA of boot file
bi_len  dd 0            ; len of boot file
bi_csum dd 0
bi_reserved times 10 dd 0
{% endhighlight %}

When the BIOS loads the bootloader, it puts the disk number of the disk where we found in into the `dl` register. We add that to our DAP now. We write the address in memory of the DAP itself to the `si` register.

We can now fire a BIOS interupt ([INT 13h AH=42](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH.3D42h:_Extended_Read_Sectors_From_Drive)) that will read from the address defined by the DAP packet.

{% highlight nasm %}
mov si, dap           ; Put the address of the dap in si.
mov dl, [boot_drive]  ; Put the boot drive # in dl.
mov ah, 0x42
int 0x13              ; Fire BIOS interrupt 13h AH=42 to read from the boot drive.
{% endhighlight %}

### 5. Load the kernel into memory

Now that we've loaded the address of the kernel, we're ready to use it the load the kernel itself into memory.

We overrwrite our DAP with the new address from which to read. We also indicate in the DAP that the kernel should be written to memory address 0x8000 - we'll jump there when we're done booting.

### 6. Enter protected mode

The i386 is capable of operating in two main modes: "real mode" and "protected mode"

In real mode, for several reasons, we only have access to the lower 1MB of memory.  Addresses above 1MB (20 bits or 5 hex digits) will either be truncated or cause an exception.

Additionally, we use 16-bit registers in real mode by default. In order to represent 20-bit addresses with 16-bit registers, we rely on logical addressing. Complete addresses are formed by combining the values in a segment register and an offset.

In protected mode we use 32-bit registers by default, so we have access to the entire 32-bit address space. That means a single 32-bit register can refer to any accessible location in memory.

The bootloader begins executing in real mode and is responsible for making the switch to protected mode. There are three steps:

1. Enable a20, the 21st line of the address bus. This gives us access to all 32 lines of the address bus.
2. Load the [Global Descriptor Table](http://www.osdever.net/bkerndev/Docs/gdt.htm) (GDT), which we defined previously in our assembly. The GDT defines base access privileges for certain parts of memory. Like the Linux kernel, we will eventually use paging to manage access privileges.
3. Set the first bit of CR0 to 1. This tells the processor to use protected mode from now on.

{% highlight nasm %}
call enable_A20    ; Allow use of all address lines on the address bus.
lgdt [GDT]         ; Load the global descriptor table, defined previously.
mov eax, cr0       ;
or al, 1           ; 
mov cr0, eax       ; Set the first bit of CR0 to 1.
{% endhighlight %}

In `ax`, `ds`, `ss`, and `es` we store offsets into the global descriptor table. The offsets point to permissions for reading, writing, and executing the code, data, and stack.

### 7. Set the stack and jump to kernel main

{% highlight nasm %}
mov esp, 0x6000      ; data stack grows down
mov eax, 0x8000      ; jump to kernel main!
call eax
{% endhighlight %}

After this `call`, the kernel code is executing.  The bootloader has set up enough scaffolding that the kernel can be written almost entirely in C.
