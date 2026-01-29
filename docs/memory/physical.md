# Free RAM Retrieval

After the hardware has been initialised by the bootloader (GRUB), control is returned to the kernel at the address pointed to via the **start** label.\
On return to the **start** label, the ebx register is loaded with an address located in the first MB of memory.\
This address points to the multiboot structure which contains information about the hardware.

Most notably, the Multiboot 1 structure returns a field with an address to a table containing **multiboot_mmap_entry** structures.\
These structures represent the current memory blocks after initialisation.\
The address may be accessed via the **mmap_addr** field and the table is of size **mmap_length**.

Using the **cos_mmap_init** function located in **src/manager_initalisation.c** in the MemoryManagement module, structures are enumerated from the table.
The type field in the **multiboot_mmap_entry** structure allows the kernel to retrieve the memory blocks marked as free.

# Basic Memory Manager Initialisation

Prior to its entry into cos_mmap_init, the kernel parsed the entirety of the multiboot structure and retrieved reserved memory regions that GRUB allocated according to the .multiboot section of the kernel. 
From the enumerated blocks, the subroutine remove_privileged_addresses will remove the memory regions that can't be used as free memory by the kernel (such as the framebuffer region for example).

Once these regions are sanitized of values that are decided at boot time, invalid blocks that are within these memory regions and well documented, will be filtered out during its second and third subroutines.  

## Free Block Claiming From Kernel Page Table

The **claim_free_kernel_blocks** function is called to extract any remaining usable memory blocks from the kernel’s page tables.\
This function is located inside the **src/manager_initalisation.c**.

Each memory block is called a page frame and accounts for exactly 4096 bytes of memory each.   
These 4kb memory blocks are defined via the **page_frame_entry_t** structure and the linked lists that were initialised points to **page_frame_entry_t** entries.   
The **available_process_frames** list is the one loaded with free page frames.   
Although it is known that huge pages can be used (2MB and 1GB pages), the implementation currently chosen as the MemoryManager exclusively uses 4kb frames.

At this point, paging is already active and the kernel page tables are set up.\
The kernel’s level-4 page table (PML4) is stored in the **kernel_page_table** field of the **cos_memory_properties_t** structure passed to this function.\
To access all mapped regions of physical memory, the function begins its traversal at entry 511 of the PML4.\
This corresponds to the canonical high half of the virtual address space (starting at 0xffff800000000000), where the kernel resides.

## Page Table Traversal

The function walks through all levels of the page table hierarchy:

    PML4 Entry (Level 4) → kernel_page_table->entries[511]
    PDPT (Level 3) → kernel_page_table_level_3
    PD (Level 2) → kernel_page_table_level_2
    PT (Level 1) → kernel_page_table_level_1

At each level, it verifies that the current entry is valid (present and either read_write or pmd_address as required).\
If any check fails, that branch of the address space is skipped.

## Allocating Page Frame Structures

The function attempts to populate **page_frame_entry_t** structures to represent each reclaimed 4KB memory block.\
These are stored in memory allocated at runtime from free memory blocks — tracked by the **next_kern_addr_page_frames_list** field of the memory properties.

There are two cases:

- Start of a New Block:
    - If no structures exist yet (last_page_frame_list_structure_ptr == NULL) or the end of the block is reached (checked via is_end_of_block), a new 4KB memory block is reserved and zeroed.
    - The first page_frame_entry_t structure inside this block is added to the allocated frames list, and the bookkeeping pointer (last_page_frame_list_structure_ptr) is updated.

- Within Current Block:
    - If space remains in the current 4KB page-frame-list block, a new page_frame_entry_t is added immediately after the previous one.\
    This entry is added to the available frames list, and the list tail pointer is updated.

The physical address of the memory block is stored in the **.address** field of the **page_frame_entry_t**.\
Once recorded, the original mapping in the page table is invalidated by setting its PTE to 0.

## Claiming the remaining RAM

The function **claim_free_page_frames** is used to claim every other page frames present in RAM by adding more **page_frame_entry_t** to the linked list of **available_page_frames**.  
It uses the same process to expand this list as the one used for reclaiming unused mapped kernel frames.    
We calculated that this process can never run out of frames to map the entire free space since it reuses already claimed free frames.   
If, for any reason, the kernel runs out of space for allocating page frame structures, it prints a red warning to the terminal using **cos_term_set_color**, and **out_of_memory_warning** is triggered.

## Kernel Process Allocation

Due to its design, the MemoryManager has to ensure that at least one process representing the kernel is properly created during this step.  
When the kernel blocks are reclaimed, it allocates memory for a **cos_process_t** structure.    
The reason it is done this way is due to the fact that the malloc reimplementation and the scheduler were being developed at the same time while refactoring the mechanisms of the MemoryManager.   
Now, we could have this first process created using **cos_malloc** instead.

External Documentation

* [Low Memory Extraction](https://wiki.osdev.org/Detecting_Memory_(x86))
* [Understanding x86_64 paging](https://zolutal.github.io/understanding-paging/)
