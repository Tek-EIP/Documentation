# Documentation of the Compatible Operating System

## Project layout

This documentation lives within the core of the project whereas Git Submodules were made to handle several aspects of the project namely:
* [The Memory Manager (Handles memory ranges, mappings and a part of its scheduling)](./Memory.md)
* [The FileSystem (EXT4 reimplementation)](./Filesystem.md)
* The ELF Binary parser
* The WinPE Binary parser

[Initialisation routines](./Kernel.md) were left within the core.



## Build Commands

To build the project as an ISO file called cos.iso:

    git submodule update --init --recursive
    make 

To build the Unit Tests:

    make test_run
