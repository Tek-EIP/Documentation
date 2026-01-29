# Bare Metal Support

## Requirements

This PoC can only run on bare-metal hardware with the following requirements:
- An Intel IDE Controller with support for Legacy Mode or that can be switched into Legacy Mode at runtime and booted loaded in ATA Mode.
- A drive containing an MBR with Partition 0 being formatted into the minimum EXT4 reimplementation from this project
- A Legacy BIOS or a UEFI with support for CSM Mode.
- A CPU with an x86_64 architecture with support for SSE/SSE2/SSE3/SSSE3 (this should already be satisfied in the CPU is x86_64 anyway).

If one of these requirements cannot be satisfied, the OS should either crash as soon as it is launched or state that the IDE Controller is unsupported and ask the user to forcibly shut the computer down.

To achieve boot on real hardware, we had to somewhat hack it together on a test machine by talking to its IDE Controller using the PCI specification and reading the MBR of the drive to locate the filesystem.

## Booting the IDE Controller in ATA Mode.

On the test machine we had, Coreboot with the Tianocore EDK2 payload was installed on it.   
Of note is that this payload is a UEFI Class 3 which doesn't support CSM Mode and forced the IDE Controller in AHCI Mode instead of ATA Mode.   
This was an issue because only 48bit PIO Mode Read/Write is supported by this kernel and therefore required a drive loaded in ATA Mode.   
Using SeaBIOS as a payload instead provided the ATA Mode required.

## Booting the kernel

Ventoy was used to load an ISO of this kernel from USB.  
What we didn't know was that if Ventoy was booted in UEFI Mode, none of the ISO it will load itself could be booted in BIOS Mode.     
Using the SeaBIOS payload also resolved this issue.

## PCI Communication

### Polling for the IDE Controller

A fully documented way to use PCI devices is available on [OSDev](https://wiki.osdev.org/PCI) and most of the following information were gathered thanks to this wiki.

As far as we understand, a PCI bus allows several devices to interact with one another in a standardized manner.  
More importantly, the OS can enumerate these devices using the PCI specification.   
While there are two ways to access a device, the one using port 0xCF8 and port 0xCFC seems to be preferred.  

Port 0xCF8 represents the CONFIG_ADDRESS and port 0xCFC is its CONFIG_DATA.
One PCI address is constructed in the following manner:
- Bit 31 is an Enable Flag (Unused for this PoC)
- Bits 30-24 is Reserved space
- Bits 23-16 is the selected PCI bus
- Bits 15-11 is the device number
- Bits 10-8 is the function number
- Bits 7-0 is an offset.

A macro was created to return a valid CONFIG_ADDRESS.   
N.B: One may notice that the PCI_ADDR masks its offset with value 0xFC (0b11111100).    
According to the specification, the offset is always a multiple of 4.

```
#define PCI_ADDR(bus, device, func, offset) (1U << 31) | ((uint32_t)bus << 16) | ((uint32_t)device << 11) | ((uint32_t)func << 8) | (offset & 0xFC)
```

The PCI specification requires a Configuration Space that is 256 bytes-long and with each byte representing one PCI bus.  
Each bus can accept up to 32 devices.    
Each device can expose up to 8 functions.   
The maximum number of allowed PCI devices is therefore 256 * 32 * 8, hence the code written here:

```
Code from src/kernel/pci.c

    for (uint16_t bus = 0; bus < 256; ++bus) {
        for (uint8_t device = 0; device < 32; ++device) {
            for (uint8_t function = 0; function < 8; ++function) {
```

Once the address is created, the PCI Configuration Space is queried via port 0xCF8 and returns a 32bit value in 0xCFC.  
While the values returned in 0xCFC are device specific, the specification requires all devices to return a common header which is found in the OSDev documentation.
The values that we needed were:
- The Vendor ID (16bit value located in Bits 15-0 at offset 0x0 of the common header)
- The Class Code (8bit value located in Bits 31-24 at offset 0x8 of the common header)
- The SubClass Code (8bit value located in Bits 31-24 at offset 0x8 of the common header)
- The Prog_IF Bit (8bit value located in Bits 31-24 at offset 0x8 of the common header)

Using these value, we were able to determine which device was the Intel IDE Controller with the following algorithm.    

The specification states that a device returning the value 0xFFFF (VENDOR_INVALID) is always invalid, so it is skipped.
```
Code from src/kernel/pci.c

            vendor = pci_config_read16(bus, device, function, 0x00);
                if (vendor == VENDOR_INVALID)
                    continue;
```

A valid Intel IDE Controller has the following characteristics:

```
Code from src/kernel/pci.c

    #define VENDOR_INTEL 0x8086
    #define PCI_CLASS_MASS_STORAGE 0x01
    #define PCI_SUBCLASS_IDE 0x01
```

By polling each device, we eventually found the one we wanted to communicate with.

```
Code from src/kernel/pci.c

                if (vendor == VENDOR_INTEL && class_code == PCI_CLASS_MASS_STORAGE && subclass == PCI_SUBCLASS_IDE) {

                    cos_printf("Found Intel IDE Controller at %x:%x.%d\n", bus, device, function);
                    cos_printf("Vendor ID: 0x%X\n", vendor);
                    cos_printf("Device ID: 0x%X\n", device_id);
```

### PROG_IF Byte

An Intel IDE Controller can accept up to 4 drives on two channels called the Master and Slave respectively.     
There are two ways to communicate with the Master and Slave channels in ATA Mode; one using the Legacy/ISA Mode (which we had implemented) and the other being the PCI Native mode.    
By default, this Controller was in PCI Native Mode.    
However, depending on the PROG_IF (Programmable Interface byte) value listed in its PCI Header, it is possible to switch either one, both or none of these channels in Legacy Mode.

If bit 0 and 2 of the PROG_IF byte are set, then both channels are in PCI Mode.
If bit 1 and 3 of the PROG_IF byte are set, then either one, both or none of the channels may be switched to Legacy Mode.

Bits 0 and 2 of the PROG_IF byte represent the Master and Slave's Legacy Mode states respectively.
By flipping both to 0 and writing the modified PROG_IF byte value into the PCI Header via the 0xCFC Port, the hardware may accept or discard this change because the PROG_IF byte is supposed to be read only.
If the value is writeable, the Intel IDE Controller will change to Legacy Mode.

In our case, the hardware returned the 4 lower bits in a set state.   
By flipping bit 0 and 2 to 0, we managed to force the IDE Controller into Legacy Mode, allowing the kernel access to the drive with 48bit PIO Mode Read and Write.

```
Code from src/kernel/pci.c

                    if (!(prog_if & 0b101)) {
                        cos_printf("Both channels of the controller are already in Legacy Mode.\n");
                        return true;
                    }
                    if ((prog_if & 0b1010)) {

                        cos_printf("The disk controller reported that it can be switched to Legacy Mode.\n");
                        cos_printf("Trying to switch the controller to Legacy Mode...\n");

                        pci_config_write8(bus, device, function, 0x09, prog_if & ~((1 << 0) | (1 << 2)));

                        prog_if = pci_config_read8(bus, device, function, 0x09);
                        if ((prog_if & ((1 << 0) | (1 << 2))) == 0) {
                            cos_printf("The drive controller was successfully switched to Legacy Mode.\n");
                            return true;
                        } else {
                            cos_printf("The drive controller state hasn't changed and failed to switch to Legacy Mode.\n");
                            cos_printf("You may turn off your computer as your system is in an unsupported state.\n");
                            cos_printf("To do so, press the Power Button until the screen turns off.");
                            return false;
                        }
                    }
```

### MBR Parsing

After the drive finishes its initialisation to prepare 48bit PIO Mode Read/Write, it becomes possible to read the contents of its MBR.   
The [MBR (Master Boot Record)](https://wiki.osdev.org/MBR_(x86)) is written in the first 512 byte of the disk and describes where each partition is located on the disk.   

To parse it, it is first required to read its last 2 bytes to check if the value 0xAA55 exists.     
If it exists, it means that an MBR is probably written in the first 512 bytes of the drive.     
In case this drive is formatted with a GPT partition scheme instead, it is required to check the beginning of the next 512 bytes for the string "EFI PART".     
If this string exists, the OS states that it cannot locate the partition because this partition scheme is unsupported.  

```
Code from FileSystem/src/wrappers.c

    if (*(uint16_t *)(&mbr[510]) == 0xAA55) {
        read_byte((uint16_t *)gpt, 1, 1);
        if (!cos_strncmp("EFI PART", (const char *)gpt, 8)) {
            cos_printf("GPT Partition Parsing hasn't yet been implemented.\n");
            cos_printf("You may shutdown your computer by pressing down the Power Button until the screen turns off.\n");
            return false;
        }

```

If it doesn't, we need to enumerate the MBR's entries.  

The MBR can only hold up to 4 partitions and its entries start at offset 0x1BE (in the first 512 bytes of the disk).
Each entry is 16 byte long and each follows the same format with the most important information being:
- Its Partition Type (Offset 0x4)
- Its LBA of partition start (Offset 0x8)

A partition type equal to 0x83 means that it is a Linux Partition.      

```
Code from FileSystem/src/wrappers.c

        for (uint8_t i = 0; i < 4; ++i) {
            if (((uint8_t *)mbr)[0x1BE + i * 16 + 4] == 0x83) {
                lba_start = *(uint32_t *)(&mbr[0x1BE + i * 16 + 8]);
                cos_printf("New Linux EXT4 Partition may have been found, lba_start is now: %d\n", lba_start);
                if (check_for_valid_ext4_partition()) {
                    cos_printf("A valid EXT4 Partition was found on your system.\n");
                    return true;
                }
            }
        }
```

Once the starting LBA of the partition is found, several checks are made on its filesystem to ensure that the kernel communicate with a valid EXT4 partition.

```
Code from FileSystem/src/wrappers.c

static bool check_for_valid_ext4_partition(void)
{
    superblock_t sb;

    memset(&sb, 0, sizeof(sb));
    read_superblock(&sb);
    if (sb.s_magic != SUPERBLOCK_MAGIC) {
        cos_printf("This disk image hasn't been formatted as an EXT4 filesystem.\n");
        return false;
    } else if (sb.s_log_block_size != 2) {
        cos_printf("This EXT4 filesystem isn't formatted as a 4096 block sized filesystem.\n");
        return false;
    } else if (sb.s_inode_size != DEFAULT_INODE_SIZE) {
        cos_printf("The EXT4 fileystem inode size is unsupported (not 256 bytes long).\n");
        return false;
    } else if ((((1024 << sb.s_log_block_size) * sb.s_blocks_per_group) / sb.s_inodes_per_group) != DEFAULT_INODE_RATIO) {
        cos_printf("The EXT4 filesystem isn't formatted with an inode ratio of 16384 per group.\n");
        return false;
    } else
        cos_printf("The EXT4 filesystem was correctly formatted and should be ready for use.\n");
    return true;
}
```
