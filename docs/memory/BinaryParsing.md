# Binary Launching

## Common Parsing Structure

The kernel is in charge of launching a new process.     
To do so, a common structure was created to retrieve information about ELF64 and WinPE 32+ binaries.

```
typedef struct BinFileData
{
    uint64_t entry_point;
    uint64_t bin_load_address;
    uint64_t stack_size;
    uint64_t heap_size;
    uint64_t number_of_sections;
    uint64_t linkage_table_pointer; //To bind imported functions to the binary in memory, its Import Address Tables must be rewritten.
    BinFileImports_t *imported_libraries;
    BinFileSection_t *sections;
} BinFileData_t;

```

- The entry_point is the virtual address of the first instruction of the executable.
- The bin_load_address is the base address of this binary.
- The stack_size and heap_size give information to the kernel regarding the size it should allocate for a program (WinPE binaries share this information).
- The number_of_sections is the total number of sections that should be mapped in memory.
- The linkage_table_pointer is the virtual address of the table used to dynamically link imported symbols.
- The imported_libraries array contains information about shared libraries used by the executable.
- The sections array contains information regarding each section of the binary.

From the imported symbols information are only kept:
- The name of the shared library.
- The imported symbols' names or ids into the shared library.

```
typedef struct BinFileFunction
{
    char *name;
    size_t hint_or_ordinal;
} BinFileFunction_t;

typedef struct BinFileLibrary
{
    size_t first_thunk;
    char *name;
    BinFileFunction_t *required_functions;
} BinFileImports_t;
```

From the sections information are only kept:
- The size of a section
- The virtual address it is meant to be located at.
- The offset of the section into the binary

```
typedef struct BinFileSection
{
    size_t size;
    size_t offset;
    uint64_t address;
} BinFileSection_t;
```

## WinPE Parsing

### PE Signature and COFF Header

According to the [WinPE Format Specification](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#ms-dos-stub-image-only), the binary file begins with the MS-DOS Stub.     
At byte 0x3C, the WinPE Signature is inserted ("PE\0\0").   

```
    const uint16_t address = (binary[60] & 0xFF) | (binary[61] & 0xFF) << 8;
    const size_t current_offset = address + 4;

    if (cos_strcmp((char *)&binary[address], "PE\0\0"))
```

Right after this signature, a COFF Header following the COFF Specification should be present.   
From this Header, it is required to check whether the file contains an Optional Header.
If the WinPE binary is an EXEcutable Image, then the field SizeOfOptionalHeader is non 0.      

```
    const CoffHeader *coff_header = (CoffHeader *)&binary[current_offset];

    //The specification states that DLLs and executable Images have this header.
    if (!coff_header->SizeOfOptionalHeader)
        return NULL;
```

Finally, we must check the Machine Field of this Header for the value IMAGE_FILE_MACHINE_AMD64 (0x8664) for WinPE x64 support, which is what this project requires.     

```
    if (coff_header->Machine == (uint16_t)0x8664)
        return windows64_fill_bin_file_data;
```

### Optional Header

The Optional Header contains information regarding the program itself.      
The ones needed to load the program in memory and ensure its execution are retrieved here:
```
    bin_file_data->bin_load_address = optional_header->ImageBase;
    bin_file_data->entry_point = optional_header->AddressOfEntryPoint + bin_file_data->bin_load_address;
    bin_file_data->stack_size = optional_header->SizeOfStackReserve;
    bin_file_data->heap_size = optional_header->SizeOfHeapReserve;
    bin_file_data->number_of_sections = coff_header->NumberOfSections;
    bin_file_data->sections = cos_malloc(sizeof(BinFileSection_t) * (coff_header->NumberOfSections + 1));
    memset(bin_file_data->sections, 0, sizeof(BinFileSection_t) * (coff_header->NumberOfSections + 1));
    windows64_fill_section_data(bin_file_data, coff_header, section_headers);
```

- The ImageBase corresponds to the Base Virtual Address of the WinPE Executable (Located in the lower half of memory).
- The AddressOfEntryPoint points to the OFFSET in the .text section where the first instruction to be executed is located.    
The Base Address is added to this value 
- SizeOfStackReserve and StackOfHeapReserve give the size expected for the process' stack and heap respectively.
- As the Section Table doesn't end with a NULL pointer, the NumberOfSections is given in the Optional Header. 

### Section Table

A Section Table is appended to the Optional Header and each entry is a structure of exactly 40 bytes.
The windows64_fill_section_data extracts, for each entry:
- The VirtualAddress the Section is expected to be loaded at.
- The offset to the start of the section in the binary, identified by the PointerToRawData field.
- The size of the section, identified by the SizeOfRawData field.

```
static void windows64_fill_section_data(const BinFileData_t *bin_file_data, const CoffHeader *coff_header, const SectionHeader_t *section_headers)
{
    for (size_t i = 0; i < coff_header->NumberOfSections; ++i)
    {
        const BinFileSection_t section = {
            .size = section_headers[i].SizeOfRawData,
            .address = section_headers[i].VirtualAddress + bin_file_data->bin_load_address,
            .offset = section_headers[i].PointerToRawData
        };

        bin_file_data->sections[i] = section;
    }
}
```

### Import Directory Table (IDT)

In the Optional Header, one may find the Optional Header Data Directories.      
Amongst them, always located at index 1 is an entry for the Import Directory Table.
As reference, this table may not exist and thus the entry may be empty.     
If it does exist, the entry is loaded with an offset in the binary to the Import Directory Table, identified by the field VirtualAddress.   

```
    if (optional_header->DataDirectory[1].VirtualAddress && optional_header->DataDirectory[1].Size)
```

N.B: Before checking for the existence of the table, one has to check for the existence of the entry itself by verifying the value of NumberOfRvaAndSizes in the Optional Header.   

The Import Directory Table references the section .idata if it exists.
By probing each section's VirtualAddress in the Section Table, one can find the PointerToRawData.

```
        uint32_t import_directory_section_header_index = 0;

        for (size_t i = 0; i < coff_header->NumberOfSections; ++i)
        {
            if (section_headers[i].VirtualAddress <= import_directory_virtual_address && section_headers[i].VirtualAddress + section_headers[i].SizeOfRawData > import_directory_virtual_address)
                import_directory_section_header_index = i;
        }
        
        const uint64_t pointer_to_raw_data = section_headers[import_directory_section_header_index].PointerToRawData;
```

The Import Directory Table is an array of ImageImportDescriptor structures, containing information related to the libraries imported by the binary.     
By leveraging the Size extracted from the Data Directory entry in the Optional Header, one can get the total number of ImageImportDirectory in the array and then enumerate them all.   
Information related to the imported functions will be stored in the BinFileData structure under the imported_libraries array.   

```
        const ImageImportDescriptor_t *image_import_descriptors = (ImageImportDescriptor_t *)&binary[pointer_to_raw_data + import_directory_virtual_address - section_headers[import_directory_section_header_index].VirtualAddress];
        const uint64_t number_of_image_import_descriptor = import_directory_size / sizeof(ImageImportDescriptor_t) - 1;

        bin_file_data->imported_libraries = cos_malloc((number_of_image_import_descriptor + 1) * sizeof(struct BinFileLibrary));
        memset(bin_file_data->imported_libraries, 0, (number_of_image_import_descriptor + 1) * sizeof(struct BinFileLibrary));
```

### Import Lookup Table (ILT)

During the enumeration of each descriptor, one can access the Import Lookup Table using the FirstThunk Field of the ImageImportDescriptor structure.    
The FirstThunk is the address that has to be replaced by the Symbols address at runtime, during the dynamic linking phase.      
For this project, the Symbol is first loaded in memory and then the dynamic linker will overwrite the FirstThunk, which will be used in the binary to call imported functions.      

Once the offset to the Import Lookup Table is found:
- The name of the library is retained.
The name is stored in the binary at an offset under the Name field of the Import Directory Table.
- The FirstThunk is retained.
- An array of entries meant for function resolution is created in the BinFileData structure.

```
            const uint64_t import_lookup_table_physical_address = pointer_to_raw_data + image_import_descriptors[i].FirstThunk - section_headers[import_directory_section_header_index].VirtualAddress;
            uint64_t *pointed_entry = (uint64_t *)((uint64_t)binary + import_lookup_table_physical_address);

            bin_file_data->imported_libraries[i].name = (char *)&binary[pointer_to_raw_data + image_import_descriptors[i].Name - section_headers[import_directory_section_header_index].VirtualAddress];
            bin_file_data->imported_libraries[i].first_thunk = image_import_descriptors[i].FirstThunk;
            bin_file_data->imported_libraries[i].required_functions = cos_malloc((get_number_of_64bit_entries(pointed_entry) + 1) * sizeof(struct BinFileFunction));

```

As the WinPE binary is a PE32+ file, each entry of the ILT is a 64bit integer.
The number of entries in the Import Lookup Table is deduced by enumerating all its entries until the NULL entry is found.

```
static uint64_t get_number_of_64bit_entries(uint64_t *pointed_entry)
{
    uint64_t *temp = pointed_entry;

    do
    {
        temp = (uint64_t *)((uint64_t)temp + 8);
    } while (*temp);
    return ((uint64_t)temp - (uint64_t)pointed_entry) / 8;
}

```

Each entry of the ILT is either:
- An ordinal, meaning an ID that references a symbol in the dynamically linked library with the same name as the ImageImportDescriptor name field.
- An offset into the WinPE binary to a Hint/Name Table.   
The Hint field is an index into the Export Name Table of this binary and the Name is a NULL-terminated string of the imported symbol.

To differentiate between either cases, one has to check value of Bit 63 of the entry.   
If it is 1, then it is an ordinal.  
If it is 0, then it is an offset to a Hint/Name Table.  

```
            for (uint64_t j = 0; *pointed_entry; ++j)
            {
                if (*pointed_entry & 0x8000000000000000)
                {
                    bin_file_data->imported_libraries[i].required_functions[j].hint_or_ordinal = *pointed_entry & 0xFFFFF;
                } else {
                    ImageImportByName_t *imported_by_name_function = (ImageImportByName_t *)&binary[pointer_to_raw_data + *pointed_entry - section_headers[import_directory_section_header_index].VirtualAddress];
                    bin_file_data->imported_libraries[i].required_functions[j].name = imported_by_name_function->Name;
                    bin_file_data->imported_libraries[i].required_functions[j].hint_or_ordinal = imported_by_name_function->Hint;
                }
                pointed_entry = (uint64_t *)((uint64_t)pointed_entry + 8);
            }
```

## Dynamic Linking

Once the binary is parsed correctly and a new process is created, the imports are dynamically resolved before it starts execution.  
The linkage table has its symbols addresses overwritten by enumerating every entry of the imported_libraries array created in the BinFileData structure.

```
    for (uint64_t i = 0; bin_file_data->imported_libraries[i].required_functions; ++i)
    {
        for (uint64_t j = 0; bin_file_data->imported_libraries[i].required_functions[j].hint_or_ordinal; ++j)
        {
            *(uint64_t *)(bin_file_data->imported_libraries[i].first_thunk + bin_file_data->bin_load_address + j * 8) = get_function_bind_address(bin_file_data->imported_libraries[i].required_functions[j].name);
        }
    }
```

Regardless of the binary type, when the ELF64/WinPE executable will try to call a function that was imported, it will retrieve the overwritten address to that function and then call it.
As this project can't provide the original libraries as part of its objectives, an address to a reimplementation is instead provided by the kernel.

```
uint64_t get_function_bind_address(const char *function_name)
{
	if (!cos_strcmp(function_name, "exit"))
		return (uint64_t)cos_exit;
	if (!cos_strcmp(function_name, "write"))
		return (uint64_t)cos_write;
    if (!cos_strcmp(function_name, "ExitProcess"))
        return (uint64_t)fake_exitprocess;
    if (!cos_strcmp(function_name, "GetStdHandle"))
        return (uint64_t)fake_getstdhandle;
    if (!cos_strcmp(function_name, "WriteConsoleA"))
        return (uint64_t)fake_cos_write;
    return 0;
}
```

## Binary Loading and ABI Differences

Once the New Process is added to the list of new processes, the kernel allocates memory ranges consistent with the ones retrieved by the Linux/WindowsParser and commits the content of the binary in memory.    
Due to this kernel being built from an ELF64 compliant binary, 64bit Linux binaries only needs to resolve their dynamic linking to be executed properly.    
However, in the case of a WinPE Binary, a translation algorithm is required to pass the arguments to this kernel's function resolved at runtime.

As an ELF64 compliant kernel, it follows the System V ABI.
As such, the first 6 arguments of every function this kernel possesses are passed to the RDI, RSI, RDX, RCX, R8 and R9 registers respectively with further arguments being pushed to the stack instead.     
The [Windows X64 ABI](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions?view=msvc-170#x64-register-usage) will store the first 4 arguments in RCX, RDX, R8 and R9 respectively with further arguments being pushed to the stack instead.

The macro WINDOWS_ARGUMENTS_REARRANGE retrieves the arguments of the dynamically linked function expected by the binary and an ELF64 function pointer to the equivalent function.   
This function pointer is called once every argument is in a System V compliant state.   
Upon return, every register is restored to its prior state.

```
#define WINDOWS_ARGUMENTS_REARRANGE(function, number_arguments) \
    ({ \
    __asm__ __volatile__( \
            "cld\n" \
			"pop %%rdx\n" \
    		"push %%rdi\n" \
    		"push %%rsi\n" \
    		"mov %%rcx, %%rdi\n" \
    		"mov %%rdx, %%rsi\n" \
    		"mov %%r8, %%rdx\n" \
    		"mov %%r9, %%rcx\n" \
			"mov %[nb], %%r9\n" \
			"cmp $5, %%r9\n" \
			"jb 2f\n" \
			"je 1f\n" \
			"sub $6, %%r9\n" \
			"je 4f\n" \
		"3:\n" \
			"push 24(%%rbp, %%r9, 8)\n" \
			"dec %%r9\n" \
			"jne 3b\n" \
		"4:\n" \
			"mov 24(%%rbp), %%r9\n" \
		"1:\n" \
			"mov 16(%%rbp), %%r8\n" \
		"2:\n" \
			"call *%[input_func]\n" \
			"mov %[nb], %%r9\n" \
			"sub $6, %%r9\n" \
			"jbe 6f\n" \
		"5:\n" \
			"pop %%r8\n" \
			"dec %%r9\n" \
			"jne 5b\n" \
		"6:\n" \
			"pop %%rsi\n" \
			"pop %%rdi\n" \
			"cld\n" \
		: \
        : [input_func] "r"(function), [nb] "i"(number_arguments) \
		: "rdi", "rsi", "rdx", "rcx", "r9", "r8", "rax" \
    ); \
})
```

N.B: One may notice that making use of a macro implies that the system will have to allocate space for the macro's arguments in the prolog of a function making use of it.   
Due to the number of registers clobbered by the system, it seems that GCC will choose to make use of RAX, RDX and R10 to be used for this initialisation state in every cases.      
As such, RDX may be modified before entry into the __asm__  statement of the macro.      
A temporary way to "patch" this, was to push the state of RDX to the stack prior to the macro executing and then be subsequently restored by the pop instruction present at the beginning of the macro.      
Opening this example in Ghidra shows that the __asm__ statement takes priority on the function's prolog.

```
void fake_cos_write(void)
{
	__asm__ __volatile__("push %%rdx\n" :::);
	WINDOWS_ARGUMENTS_REARRANGE(cos_write, 3);
}
```
