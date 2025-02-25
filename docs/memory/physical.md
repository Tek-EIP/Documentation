# Free RAM Retrieval

After the hardware has been initialised by the bootloader (GRUB), control is returned to the kernel at the address pointed to via the **start** label.\
On return to the **start** label, the ebx register is loaded with an address located in the first MB of memory.\
This address points to the multiboot structure which contains information about the hardware.

Most notably, the Multiboot 1 structure returns a field with an address to a table containing **multiboot_mmap_entry** structures.\
These structures represent the current memory blocks after initialisation.\
The address may be accessed via the **mmap_addr** field and the table is of size **mmap_length**.

Using the **cos_init_memory** function located in **src/kernel/memory/mmap_init.c** in the main COS repository, structures are enumerated from the table.
The type field in the **multiboot_mmap_entry** structure allows the kernel to retrieve the memory blocks marked as free.

# Basic Memory Manager Initialisation

The **initialise_memory_manager_addressing** function of the MemoryManagement module is called by the COS kernel during the final stages of the **cos_init_memory** function.\
Calling this function will initialise two linked lists with the address passed as a parameter.

The kernel passes the **kernel_end_address** variable as its parameter and it represents the first address in memory after kernel data and code.\
This address is retrieved from the **linker.ld** file located at the root of the main COS repository and it points to a memory area inside a free memory block.

The kernel then calls the function **order_free_physical_blocks** after **initialise_memory_manager_addressing** returns.\
It will pass the free memory block addresses and block sizes that were collected by the cos_init_memory function to the MemoryManagement module.\
Free memory blocks will be divided into arbitrary 64kb memory blocks,

These 64kb memory blocks are defined via the **page_frame_entry_t** structure and the linked lists that were initialised points to **page_frame_entry_t** entries.\
The **available_process_frames** list is the one loaded with 64kb blocks.

However, during the division of free memory areas, the final 64kb block may not be of size 64kb.
If this is the case, the block will instead be added to the **available_process_frames_short** list.

This division will allow the kernel to easily allocate page frames for programs that will be loaded in memory.\
A page frame is a block of 4kb of memory used within the paging mechanism of the CPU.

# Retrieving the virtual address (Incomplete)

After successful initialisation of the MemoryManagement module, a new virtual address range may be retrieved via the **get_virtual_address_range** function.

In order to execute a binary file, its .text section must be loaded in memory first.\
To do so, the kernel has to create a page table for the program.\
As paging is enabled during hardware initialisation, the pointer to this table must be passed to the cr3 register.

**THE FOLLOWING IS UNFINISHED**

In its current state, the MemoryManagement module retrieves the address of the kernel page table contained within the cr3 register and uses it as the page table of the executable.\
The table is explored recursively with the **discover_free_address** function until free entries are found to store the program in memory.

The location within the page table can be transformed into a virtual address as virtual addresses are indexes within the page table (cf. **Kernel Page Table Definition** section).

The **allocate_page_table_entries_at_launch** currently creates the virtual address.\
The switch statement uses the special fall through comment to remove gcc warnings at compile time.\
Depending on the depth of the **discover_free_address** function, the switch statement is supposed to OR the index within the page table level in the **virt_addr** variable and then shift the index to the right following the virtual address format.

At the same time, it updates the kernel page table to initialise new pages that may be accessed by the program requesting memory.

When this function returns, it currently initialises 2MB of memory.\
As this functionality is incomplete, a full rework is planned.

Once the virtual address is made, the pointer to the kernel page table is moved into cr3 again.\
Although it is unnecessary for now, this part of the MemoryManagement will be used for execution of a binary with its own page table instead of the kernel's.

External Documentation

* [Low Memory Extraction](https://wiki.osdev.org/Detecting_Memory_(x86))
* [Understanding x86_64 paging](https://zolutal.github.io/understanding-paging/)
