# Scheduling

To complete this project, scheduling was supposed to be a core functionality.   
As it was made near the end of the project, it is handled using a simple round-robin algorithm without priority.

## Process creation

The following structure describes a process in the project.
```
Code from MemoryManagement/include/cos_memory_management.h

typedef struct cos_process {
    uint64_t id;
    uint64_t privilege_level;
    uint64_t process_start;
    uint64_t quantum;
    page_frame_entry_t *allocated_page_reserved;
    page_frame_entry_t *allocated_process_memory;

    uint64_t stack_top;
    uint64_t kernel_stack_top;

    uint64_t heap_start;
    uint64_t heap_end;

    cos_cpu_state_t register_states;
    page_table_t *page_table;
    struct cos_process *next_proc;
    struct cos_process *prev_proc;
    char *command;
}  cos_process_t;

```

* The process id is an integer incremented by 1 each time a new process starts.     
Ideally, it should've been defined in accordance with the Linux Kernel way of calculating process ids.      
Process 0 is always the kernel itself.
* The privilege_level can only be either 0 or 3, corresponding to the x86_64 Privilege Ring Levels for Supervisor and User privilege respectively.
* The process_start integer is the time the process was first launched.   
In Qemu, the kernel starts at the time the process opening the Qemu session was first started.    
For instance, if Qemu is opened in a terminal that was running on the host for 5minutes, the guest will state that the kernel has been running for 5 minutes as well.
* The quantum is a constant representing the time that a process is allowed to run.
* The linked list of allocated_page_reserved contains the nodes of the paging tree for this process.   
The root of the tree uses entry 511 for kernel page access and entry 510 for its recursive paging.
The linked list of allocated_process_memory contains page frames allocated by malloc as well as mapped regions of memory during the process runtime.    
* The stack_top is the memory address pointing to the top address of the process stack.   
Contrary to what the name might imply, it is the bottom of its stack because the stack grows downwards.
For user processes, this stack starts at the maximum virtual address in the lower half of memory.
* The kernel_stack_top is the memory address pointing to the top address of the process' kernel stack.    
Whenever a syscall is made, the process residing in Ring 3 has to kernel pages to handle it.  
As such, every process has its own kernel stack that is switched with its user stack at runtime by the kernel.
* The heap_start and heap_end pointers define a virtually contiguous memory region which gives a process a heap and which is allowed to be expanded if a process requires it.     
It is controlled exclusively by malloc and free.
The heap starts resides at the lowest address after the .text and .data/.rodata sections.     
Compared to the stack, it grows upwards.
* The register_states are saved whenever its quantum of execution ends.
* The page_table is a pointer to the root node of the paging tree for this process.   
It is required because it must be inserted in CR3 for the CPU to make use of the process' paging tree.
* Each process is added to a double circular linked list of running processes for which the next_proc pointer points to the next process in the list and the prev_proc pointer to the one before.
* The command that started the process is kept as a string displayed by the ps command of the project.

At creation time, the new process is given a privilege level of 3, its stack is created and a new kernel stack is added to its structure.   
The stack size is retrieved from the ELF/WinPE header and passed to this function.      
The proc_command argument is the command string.
```
    cos_process_t *user_proc = create_new_process(3, bin_file_data->stack_size, proc_command);
```
The user stack is loaded with an iret_frame with the following parameters:
```
    user_proc->register_states.rbp = user_proc->stack_top;
    user_proc->register_states.cr3 = (uint64_t)virtual_to_physical_address(user_proc->page_table, (uint64_t)user_proc->page_table);
    user_proc->register_states.stack_frame.cs = 0x20 | 3;
    user_proc->register_states.stack_frame.flags = 0x202;
    user_proc->register_states.stack_frame.rip = bin_file_data->entry_point;
    user_proc->register_states.stack_frame.rsp = user_proc->stack_top;
    user_proc->register_states.stack_frame.ss = 0x18 | 3;
    user_proc->register_states.gs = 0x18 | 3;
    user_proc->register_states.fs = 0x18 | 3;
```

When an interrupt is made, the scheduler switches the process to the next one in the list and the iret_frame is poped from the stack and changes the values of each register accordingly.
The new Code Segment Selector points to the GDT entry containing a code segment with User (Ring 3) permissions.     
Its Segment Selector points to an entry containing a data segment with User (Ring 3) permissions.   
The entry_point is the first instruction of the program loaded in memory.   
As the FS and GS registers are the only data segment registers used by a CPU in long mode, they must be updated to point to a data segment with User (Ring 3) permissions.

After the ELF/WinPE header is parsed, we need to enumerate each sections to find the lowest possible address at which the heap can start.    
Once it is found, it is mapped inside the paging tree of the user process in the lower half region of memory.

```
    for (uint64_t i = 0; i < bin_file_data->number_of_sections; ++i)
    {
        if (!bin_file_data->sections[i].size)
            continue;
        map_process_section(bin_file_data->sections[i].address, bin_file_data->sections[i].size);
        cos_memcpy((void *)bin_file_data->sections[i].address, &binary[bin_file_data->sections[i].offset], bin_file_data->sections[i].size);
        if (highest_segment_address < bin_file_data->sections[i].size + bin_file_data->sections[i].address)
            highest_segment_address = bin_file_data->sections[i].size + bin_file_data->sections[i].address;
    }

    user_proc->heap_start = (highest_segment_address & ~0xFFF) + 0x2000;
    user_proc->heap_end = user_proc->heap_start + bin_file_data->heap_size;
    map_process_section(user_proc->heap_start, bin_file_data->heap_size);
```

## Syscalls

When a syscall is made by a user process, the syscall handler of the kernel makes use of two parts.     
The one stored in the lower half makes use of the syscall instruction and the one handling the syscall in the higher half makes use of sysret.      
To enable support for both of these instructions in Long Mode, MSR Registers must be interacted with.

In Intel 64bit mode, it is required to set the IA32_EFER Bit (Bit 0) of MSR Register 0xC0000080.    
The MSR STAR Register (0xC0000081) must be loaded with a value referencing a GDT Offset to a data and code segment such that:
- CS is loaded with the Bits 63:48 + 16 of this value
- SS is loaded with the Bits 63:48 + 8 of this value
The value 0x0013000800000000 satisfies these conditions.
The MSR LSTAR Register (0xC0000082) is loaded with the 64bit address used to pursue execution after the switch to Ring 0 is performed.

As a reminder, MSR Registers can only be interacted with using the rdmsr and wrmsr instructions.    
The ECX register is loaded with the MSR Register one wishes to access and the pair EDX:EAX is used:
- by rdmsr to retrieve the value stored in the MSR Register.
- by wrmsr to write the value stored in the EDX:EAX pair into the MSR Register.

```
void enable_syscall(void)
{
    uint64_t addr = (uint64_t)cos_syscall80;

    __asm__ __volatile__ (
        "mov $0xC0000082, %%rcx\n"
        "mov %0, %%rax\n"
        "mov %%rax, %%rdx\n"
        "shr $32, %%rdx\n"
        "wrmsr\n"

        "mov $0xC0000080, %%rcx\n"
        "rdmsr\n"
        "or $0x1, %%rax\n"
        "wrmsr\n"

        "mov $0xC0000081, %%rcx\n"
        "xor %%rax, %%rax\n"
        "mov $0x00130008, %%rdx\n"
        "wrmsr\n"

        :: "r" (addr)
        : "rax", "rcx", "rdx"
    );
}
```

To use the syscall instruction, one must preserve the R11 and RCX registers because:
- syscall stores the eflags state into R11 and stores the return address into RCX before loading RIP with the address stored in the LSTAR Register.
- sysret loads the eflags state from R11 and returns to the address loaded into RCX.

As the System V ABI uses RCX as the 4th argument passed to a function, R10 takes the role of the RCX Register instead.      
By convention, this kernel stores the syscall number in the RAX Register.

```
_cos_write:
    push	r11
    push	rcx
    mov rax, 1
    o64 syscall
    pop rcx
    pop r11
    ret
```

## Task State Segment (TSS)

When an interrupt is fired, the kernel is supposed to handle it.    
However, while the current process is a Ring 3 process, it doesn't have the permission necessary to access kernel code.     
As such, a special segment stored in the GDT is used to switch the current task to a Ring 0 task instead.   
Compared to the other GDT Segments, it is 128 bytes long because it must store a 64bit sized base address (which is ignored) as well as other parameters.    

A stack built with Ring 0 pages is placed in the RSP0 field of the TSS, which will load it as soon as a lesser privileged Ring is interrupted by the CPU.   
Upon return, the iret_frame will restore the user stack and continue execution of the current process.      

If the TSS stack is changed at runtime, the GDT has to be reloaded with the lgdt instruction.   
The ltr instruction is an offset into the GDT pointing to the Task State Segment.   

```
void reload_tss(uint64_t stack_ptr)
{
    cos_gdt_entry_tss_high_t tss_high = {};

    tss.rsp0 = (uint64_t)stack_ptr;
    tss.iopb = sizeof(tss_t) - 1;
    tss_high.base3 = ((uint64_t)&tss >> 32) & 0xFFFFFFFF;

    setup_gdt_entry(5, ((uint64_t)&tss) & 0xFFFFFFFF, sizeof(tss_t) - 1, 0x89);//PRESENT | ACCESSED | EXEC);
    cos_memcpy(&gdt64[6], &tss_high, sizeof(tss_high));
    __asm__ __volatile__ (
        "lgdt %[gdt_ptr]\n"
        "ltr %%ax\n"
        :
        : [gdt_ptr] "m" (gdt64_ptr), "a" (0x28)
    );
    reload_segments();
}
```
