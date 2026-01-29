# Streaming SIMD Extensions (SSE)

As this project is shall launch at least a Hello World in ELF format and WinPE format, we ran into an issue while trying to launch both.    
As it turns out, the compiler used instructions that are part of the SSEs to compile these Hello World binaries.    
Support was therefore added for SSE, SSE2 and SSE3.

According to [Section 11.6 of Intel's Manual Volume 3](https://cdrdv2.intel.com/v1/dl/getContent/671447), the process for initialization of SSE/SSE2/SSE3 is the following:
* Check support for these by making use of the CPUID instruction.
* Set SSE flags in the CR4 register.
* The MXCSR register controls can then be used to alter the way SSE works (this PoC doesn't make use of these functionalities).

Read [Section 10 of the Intel Manual Volume 1](https://cdrdv2.intel.com/v1/dl/getContent/671436) to learn about the MXCSR register flags.

N.B: OsDev states that CR0's flags must also be altered even though Intel's documentation doesn't document this under the SSE set up.   
Testing in Qemu shows that it doesn't seem to make any difference whatsoever for this PoC though.

The function enable_sse is stored in the main64.asm file of this project under the main COS repository.


# Compilation issues

At several points during the development of this PoC OS, we noticed several issues which stemmed from the compiler itself.  
In no particular order:

## Red Zone
    
We discovered the Red Zone while scouring documentation on [OSDev](https://osdev.wiki/wiki/Libgcc_without_red_zone) to write the project.   
The Red Zone is a 128 byte zone located under the RSP register that the compiler can allocate in a function's prologue.     
According to [Section 3.2.2 of the System V x86_64 ABI](https://gitlab.com/x86-psABIs/x86-64-ABI), this zone shall not be modified by signal or interrupt handlers.     
Instead of wasting time trying to debug invalid behaviour *which hadn't occurred yet*, we disabled this functionality instead.    
To ensure that the compiler never adds any Red Zone at compile time, one has to pass to GCC the -mno-red-zone flag.

## Default functions

Before a malloc/free implementation was provided, we made use of static arrays to test functionality of the EXT4 FileSystem Read/Write implementation.    
We discovered that once a certain static array size is reached, GCC calls memset to zero out the array.   
Thanks to this behaviour, we realized that GCC's freestanding environment may invoke functions from LibC.   
[Section 2.1 of the GNU Documentation](https://gcc.gnu.org/onlinedocs/gcc/Standards.html) states that the following must be provided by the project if LibC isn't used:
- memset
- memcpy
- memcmp
- memmove

## Stack Protection

As stated in the prior section, we made use of static arrays while building the project.    
Due to this, GCC tried to invoke __stack_chk_fail as [Stack Smashing Protection](https://osdev.wiki/wiki/Stack_Smashing_Protector) is enabled by default.   
It is meant to be a protection against buffer overflows and several GCC flags can change its behavior or outright disable it completely.    
According to the Linux Foundation, [__stack_chk_fail shall abort execution and exit the program](https://refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/baselib---stack-chk-fail-1.html).
