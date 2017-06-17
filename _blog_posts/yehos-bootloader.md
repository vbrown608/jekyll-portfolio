---
title: Building a Bootloader
date: 2017-06-17T10:20:00Z
keywords: Operating Systems, Yehos
summary: Yehos is a simple operating system built by Andrea Law, Dominic Spadacene, Mortiz Neeb, Nandaja Varma, Saul Pwanson, and Vivian Brown. It runs on the x386 processor and is designed to demonstrate fundamentals of OS development. In this post I explain how the Yehos bootloader works.
---

## What is a bootloader and why do we need it?

When you power on your computer .

While this article covers the Yehos bootloader in particular, it should demonstrate the general process of booting on x86 hardware. Our annotated bootloader.asm is available [on Github](https://github.com/domspad/yehos/blob/master/bootloader.asm).

## The boot sequence

### 1. Press power to start the BIOS

When a user presses the power button, the computer hardware fires a reset signal that starts the BIOS (Basic Input/Output System). The BIOS is a very simple operating system embedded on the same chip as the processor. 

The BIOS does some basic hardware checks to make sure the keyboard, monitor, and other devices are working.

### 2. The BIOS loads the bootloader

Next, the BIOS looks for a bootloader. It searches each storage device for a "boot signature", a special sequence of bits that indicates the presence of a bootloader. Ours is appended to the end of our bootloader like this:

```
times (512 - $ + entry - 2) db 0 ; pad boot sector with zeroes
db 0x55, 0xAA ; 2 byte boot signature
```

When the BIOS finds the boot signature, it loads the preceding block into main memory at starts executing at the beginning of that block. At this point, we're in our bootloader.

### 4. Prepare for boot

We start of with a bit of housekeeping by:

1. Turning off interrupts. We don't want a hardware interrupt to get in the way of the boot process.
2. Zeroing out all registers.
3. Seting up the boot stack by moving a known address into the stack pointer.

### 3. Get the address of the kernel

The BIOS only loaded a single sector of our kernel into memory (the bootloader). We need to load the rest of the kernel. First, we need to tell the bootloader where to find it.

Because we're storing our kernel on an ISO, we don't know before the build exactly where it will be located. To help find it later, at build time we write the address of the kernel to the first sector of the ISO.

We've populated the beginning of our bootloader with a Disk Address Packet (DAP) that contains information about loading data off the ISO into memory. Our DAP looks like this:

```
iso_boot_info:
bi_pvd  dd 16           ; LBA of primary volume descriptor
bi_file dd 0            ; LBA of boot file
bi_len  dd 0            ; len of boot file
bi_csum dd 0
bi_reserved times 10 dd 0
```

When the BIOS loads the bootloader, it puts the disk number of the disk where we found in into the `dl` register. We add that to our DAP now. We write the address in memory of the DAP itself to the `si` register.

We can now fire a BIOS interupt ([INT 13h AH=42](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH.3D42h:_Extended_Read_Sectors_From_Drive)) that will read from the address defined by the DAP packet.

```
mov si, dap           ; Put the address of the dap in si.
mov dl, [boot_drive]  ; Put the boot drive # in dl.
mov ah, 0x42
int 0x13              ; Fire BIOS interrupt 13h AH=42 to read from the boot drive.
```

### 4. Load the kernel into memory

Now that we've loaded the address of the kernel, we're ready to use it the load the kernel itself into memory.

We overrwrite our DAP with the new address from which to read. We also indicate in the DAP that the kernel should be written to 0x8000 - we'll jump there when we're done booting.

### 5. Enter protected mode

The processor is capable of operating in two different modes: "real mode" and "protected mode" 

In real mode, we only have access to the first 20 lines of the address bus (0-19). That means we can't refer to addresses over 20 bits (5 hex characters or 1MB).

Additionally, we use 16-bit registers in real mode. In order to represent 20-bit addresses with 16-bit registers, we rely on logical addressing. Complete addresses are formed by a segment register and an offset.

In protected mode, we use 32-bit registers have access to all 32 lines of the address bus. That means we can refer to any accessible location in memory with a single register.

The bootloader begins executing in real mode and is responsible for making the switch to protected mode. There are three steps:

1. Enable a20, the 21st line of the address bus. This gives us access to all 32 lines of the address bus.
2. Load the [Global Descriptor Table](http://www.osdever.net/bkerndev/Docs/gdt.htm) (GDT), which we defined previously in our assembly. The global descriptor table defines base access privileges for certain parts of memory. Like more operating systems, we will eventually use paging to manage access privileges.
3. Set the first bit of CR0 to 1. This tells the processor to use protected mode from now on.

```
call enable_A20    ; Allow use of all address lines on the address bus.
lgdt [GDT]         ; Global the global descriptor table, defined previously.
mov eax, cr0       ;
or al, 1           ; 
mov cr0, eax       ; Set the first bit of CR0 to 1.
```

In `ax`, `ds`, `ss`, and `es` we store offsets into the global descriptor table. The offsets point to permissions for reading, writing, and executing the code, data, and and stack.

### 6. Set the stack and jump to kernel main

```
mov esp, 0x6000      ; data stack grows down
mov eax, 0x8000      ; jump to kernel main!
call eax
```
