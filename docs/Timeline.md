# Project Timeline

## Context

The point of this project was to learn about OS Development on the x86_64 platform using several resources and being able to launch 64bit ELF and WinPE binaries through a simple PoC kernel.    
This document details the timeline of the project including the several hurdles we met alongside its development.   

### Timeline

* [64bit Initialisation](./initialisation/64bit_launching.md)
* [PIC Initialisation](./initialisation/idt_init.md)
* [IDT Initialisation](./initialisation/idt_init.md)
* [Physical Memory Retrieval](./memory/physical.md)
* [VGA Prompt](./kernel/prompt.md)
* [Memory Mapping](./memory/virtual.md)
* [Drive Initialisation in 48bit LBA Mode](./filesystem/asm_functions.md)
* [EXT4 FileSystem Formatter and Mounting](./filesystem/ext4_setup.md)
* [Terminal Commands to interact with the EXT4 FileSystem]()
* [Framebuffer Prompt](./kernel/prompt.md)
* Pure ASM binary launching (Test of the Memory Mapper and replaced by real formats)
* [Bare-Metal Support WIP (PCI Parsing and MBR/GPT Parsing)](./kernel/pci.md)
* [WinPE Parser](./memory/BinaryParsing.md)
* [WinPE Loading](./memory/Linux_parser.md)
* [ELF Parser](./memory/Linux_parser.md)
* [ELF Loading](./memory/Linux_parser.md)
* [PoC Scheduling](./memory/scheduling.md)

### WIP Features

The following WIP features may have uncompleted documentation.
* Graphics Support
* ACPI Tables Parsing
* [File Descriptor Functionality](FilesystemCommands.md)
