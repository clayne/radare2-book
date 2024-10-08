## Address Zero

At the beginning of the file we use to find the entrypoint for the boot and sometimes an interrupt vector table, also known as a list of pointers to functions handling specific hardware events.

Reading the code from the very beginning will teach us several things that need to be written down in our startup scripts to get proper analysis.

* Where the binary is located (base address)
* Magic header with branch to code and metadata
* The initial value for the stack pointer
* A branch to the firmware entrypoint
* Hardware initialization routines
* etc.

Boot loaders use to get all the initialization in the begining before jumping out to the main loop or logic, but microcontrollers in the other side use to have an array of pointers to functions.

Those pointers can give us some hints to know where the code is located and which addresses need to be mapped. The first pointer in this interrupt vector array is the **RESET**. Which points to the function that will be executed at boot time, or when the reset button is pressed.

Some critical systems will reset the hardware and trigger this function when there's a hardware error.

We can use the `pxw` or `pv4` commands to read those pointers. Note that `ad` will analyze the data automatically for us and resolve the pointers and structures and possible logic behind each value. But we need to have `-b 32` to take it as 32 bit dwords.

Another useful command to analyze pointers and vector interrupts is `pxr` which is a periscoped version of `px`. The same goes for `drr` that's applied for analyzing the cpu register values.

## Base address

The base address is the address in memory where the firmware is mapped when loaded by the bootloader. This is defined at compile time and specified by the memory layout linker script that is usually tied to the hardware manufacturer and SoC model.

We can specify this using the `-m` flag of radare2.

Note that there's also `-B` which is refering to the base address. It's important to understand the difference between them because -m is mapping the whole binary in a specific address, which affects the physical addressing and -B just changes the virtual address rules used by RBin when mapping the binary segments in memory.

In other words:

* Use -B for normal binaries
* Use -m for firmwares and raw dumps

As previously said, by reading the disassembly in the address 0 we can usually find out this value. Let's take this TriCore binary

```console
$ r2 -a tricore.cs -b 32 -e asm.emu=true raw.bin
[0x00000000]> pd 5
0x00000000   movh.a a2, 0x8000   ; a2=0x80000000
0x00000004   lea a2, [a2]0x3124  ; a2=0x80003124
0x00000008   nop
0x0000000a   ji a2
0x0000000c   nop
[0x00000000]> 
```

We are using the `-e asm.emu=true` option to tell radare2 to emulate the code so we don't need to manually do the computations done by the assembly and find out the final values used.

This code does:

```
- a2 = (0x8000 << 16) + 0x3124
- goto a2 // pc=0x80003124
```

The information we get here is:

* firmware is mapped at address 0x80000000
* entrypoint is located at 0x80003124

So now we can update our script to use `r2 -m 0x80000000` and set a flag in the entrypoint with `f entry0=0x80003124`

## Memory layout

Flash dumps usually contain more than just code, the memory layout may differ depending on the device and we can find different setups for the MMU depending on the time of execution.

Most of the time the code will run in real mode, which means that there are no memory protections or isolated executions of different programs. So we can assume the following blocks to be mapped in memory:

* Flash at address zero
* RAM memory (used for stack and heap)
* ROM memory (read only)
* EEPROM (slow writing, used for configurations)
* Cached memory (a copy of another region)
* Peripherals (memory mapped devices)

## Map attributes

To create the maps that hold all the memory layout needed to get proper disassembly, analysis and emulation of the firmware we will use different `o` subcommands.

Other reverse engineering tools are very limited and only allow you to define the following aspects:

* starting address
* ending address
* permissions (rwx)
* volatile (for ram)
* name

But radare2 provides many more options and ways to get your memory ready. The following options can't be found in any other reverse engineering tool out there:

* Multi layer caches
* Overlapping maps with priorities
* Multiple swapable memory banks
* Cyclic memory layouts
* Bound to Nth bits data buses
* Non-byte address spaces (7, 13 bit ...)
* Separate Code and Data
* Segmented memory
* Bake maps with data from files
* Sub-maps caching portions of other maps or files

For now we will just cover the basics which are important to understand and get right in order to understand the rest of the concepts.

## Setting up the memory

We have already mapped the firmware to the base address using the `-m` commandline flag.

But now we need to create a new map for the RAM. Let's imagine our device have 8MB of ram located at address 0x4000_0000:

```console
> on malloc://8M 0x40000000
```

What this command will do is open the uri malloc:// which opens a new file inside radare2, (the on command won't try to parse any binary header, which is unnecessary) and the 'file' part of the uri specifies the size (using the RNum library considers that it's about MegaBytes because the number ends with 'M').

The second argument tells where this file needs to be mapped in memory.

We can verify that everything worked as expected by using the `om` command like this:

```console
[0x00000000]> om
- 2 fd: 4 +0x00000000 0x80000000 - 0x087fffff r-x
* 1 fd: 3 +0x00000000 0x40000000 - 0x007fffff rwx
[0x00000000]> 
```

Naming maps is done with the `omn.` command:

```console
[0x00000000]> omn. FLASH @ entry0
[0x00000000]> omn. RAM
- 2 fd: 4 +0x00000000 0x80000000 - 0x087fffff r-x FLASH
* 1 fd: 3 +0x00000000 0x40000000 - 0x007fffff rwx RAM
[0x00000000]> 
```

We can change the permissions of each map using the `omp` command but bear in mind that we won't be able to make a page writeable if the underlying file descriptor is read-only.

To write the contents of a file inside a map we may use the `wff` command.

```console
[0x00000000]> wff?
| wf[fs] -|file  write contents of file at current offset
```
