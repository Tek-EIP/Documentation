# ELF Parser Documentation

This document describes the design and behavior of an **ELF (Executable and Linkable Format) parser**. It explains each ELF structure and section extracted by the parser, their purpose, and how they are typically interpreted at runtime.

The documentation is written to be implementation-agnostic, but it matches the behavior of a low-level C ELF parser used in an operating system or kernel-like environment.

---

## 1. Overview of ELF

ELF (Executable and Linkable Format) is the standard binary format used on Unix-like systems (Linux, BSD, etc.) for:

* Executable files
* Object files
* Shared libraries
* Core dumps

An ELF file is composed of:

1. ELF Header
2. Program Header Table (segments)
3. Section Header Table (sections)
4. Raw binary data

The parser’s role is to:

* Validate the ELF file
* Interpret headers correctly (endianness, architecture)
* Locate segments and sections
* Extract useful metadata for loading or analysis

---

## 2. ELF Header (`Elf_Ehdr`)

The **ELF Header** is located at the very beginning of the file and provides global information about the binary.

### Fields Extracted

| Field        | Description                                        |
| ------------ | -------------------------------------------------- |
| `e_ident`    | ELF magic number and metadata                      |
| `e_type`     | File type (Executable, Shared Object, Relocatable) |
| `e_machine`  | Target architecture (x86_64, ARM, etc.)            |
| `e_version`  | ELF version                                        |
| `e_entry`    | Entry point virtual address                        |
| `e_phoff`    | Program header table offset                        |
| `e_shoff`    | Section header table offset                        |
| `e_flags`    | Architecture-specific flags                        |
| `e_phnum`    | Number of program headers                          |
| `e_shnum`    | Number of section headers                          |
| `e_shstrndx` | Index of section name string table                 |

### Purpose

The ELF header tells the parser:

* How to interpret the rest of the file
* Whether the file is valid
* Where other important structures are located

---

## 3. Endianness and Word Size

The parser determines:

* **Endianness** (Little / Big endian)
* **Word size** (32-bit or 64-bit)

This affects how multi-byte values are read from the file.

Typical helper functions:

* `read16()` – Reads a 16-bit value
* `read32()` – Reads a 32-bit value
* `read64()` – Reads a 64-bit value

Correct endianness handling is critical for portability.

---

## 4. Program Header Table (`Elf_Phdr`)

The **Program Header Table** describes memory segments used at runtime.

### Common Segment Types

| Type         | Meaning                     |
| ------------ | --------------------------- |
| `PT_LOAD`    | Loadable segment            |
| `PT_DYNAMIC` | Dynamic linking information |
| `PT_INTERP`  | Program interpreter         |
| `PT_NOTE`    | Auxiliary information       |

### Fields Extracted

| Field      | Description                        |
| ---------- | ---------------------------------- |
| `p_type`   | Segment type                       |
| `p_offset` | Offset in file                     |
| `p_vaddr`  | Virtual address in memory          |
| `p_filesz` | Size in file                       |
| `p_memsz`  | Size in memory                     |
| `p_flags`  | Read / Write / Execute permissions |
| `p_align`  | Alignment constraint               |

### Purpose

The parser uses program headers to:

* Map file offsets to virtual addresses
* Load executable segments into memory
* Resolve entry points and runtime layout

---

## 5. Virtual Address to File Offset Mapping

A common helper in ELF parsers converts a **virtual address** into a **file offset**.

### Logic

1. Iterate over `PT_LOAD` segments
2. Check if the virtual address falls within the segment range
3. Convert it using:

```
file_offset = p_offset + (vaddr - p_vaddr)
```

### Purpose

This is required for:

* Resolving symbols
* Parsing dynamic sections
* Accessing runtime data stored in the file

---

## 6. Section Header Table (`Elf_Shdr`)

The **Section Header Table** describes logical sections used for linking and debugging.

### Common Section Types

| Section     | Purpose                   |
| ----------- | ------------------------- |
| `.text`     | Executable code           |
| `.data`     | Initialized data          |
| `.bss`      | Uninitialized data        |
| `.rodata`   | Read-only data            |
| `.symtab`   | Symbol table              |
| `.strtab`   | String table              |
| `.shstrtab` | Section name string table |

### Fields Extracted

| Field          | Description                           |
| -------------- | ------------------------------------- |
| `sh_name`      | Offset into section name string table |
| `sh_type`      | Section type                          |
| `sh_flags`     | Section attributes                    |
| `sh_addr`      | Virtual address                       |
| `sh_offset`    | File offset                           |
| `sh_size`      | Section size                          |
| `sh_link`      | Linked section index                  |
| `sh_info`      | Extra information                     |
| `sh_addralign` | Alignment                             |
| `sh_entsize`   | Entry size (if table)                 |

---

## 7. Section Name String Table (`.shstrtab`)

Section names are not stored directly in section headers.

Instead:

* `sh_name` is an offset
* It points into the `.shstrtab` section

### Purpose

The parser must:

1. Load `.shstrtab`
2. Resolve human-readable section names
3. Match sections like `.text`, `.data`, `.symtab`

---

## 8. Symbol Table (`.symtab` / `.dynsym`)

Symbol tables describe functions and variables.

### Symbol Entry Fields

| Field      | Description              |
| ---------- | ------------------------ |
| `st_name`  | Offset into string table |
| `st_value` | Symbol address           |
| `st_size`  | Symbol size              |
| `st_info`  | Type and binding         |
| `st_other` | Visibility               |
| `st_shndx` | Section index            |

### Purpose

Used for:

* Linking
* Debugging
* Dynamic symbol resolution

---

## 9. String Tables (`.strtab`, `.dynstr`)

String tables store:

* Symbol names
* Section names
* Dynamic linking identifiers

They are referenced by offsets, never directly.

---

## 10. Memory Management

In a kernel or embedded environment, the parser:

* Uses custom allocators
* Avoids standard libc
* Copies data manually

Typical operations:

* `malloc`-like allocation
* Explicit `memcpy`
* Manual cleanup

---

## 11. Error Handling

Common validation steps:

* Check ELF magic number (`0x7F 'E' 'L' 'F'`)
* Validate architecture
* Ensure offsets are within file bounds
* Handle missing or malformed sections

The parser should fail early and safely.

---

## 12. Summary

This ELF parser:

* Reads and validates ELF binaries
* Extracts headers, segments, and sections
* Resolves virtual addresses
* Supports symbol and string tables

Such a parser is a foundational component for:

* Operating systems
* Bootloaders
* Binary analysis tools
* Custom loaders

---

**End of Documentation**
