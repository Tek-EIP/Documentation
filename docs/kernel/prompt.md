# Command Prompt

## VGA

## Framebuffer

As GRUB is used, one may leverage multiboot options to ask the bootloader to return a specific functionality.   
As a warning though, the Multiboot Documentation states that GRUB will return invalid information if the functionality that was asked doesn't exist.   
Otherwise, if the requested functionality does exist but cannot satisfy the specified options, GRUB will return the best option available instead.  

To request a framebuffer, the Multiboot V2 specification states that a tag of type 5 must be appended to the header.    
``` 
Code from src/boot/header.asm

framebuffer_tag_start:
    ; Tag type 5 defines the framebuffer request tag.
    ; Tells GRUB we prefer getting a framebuffer of size 1920x1080 with 32bit depth.
    dw 5
    dw 0
    dd framebuffer_tag_end - framebuffer_tag_start
    dd 1920
    dd 1080
    dd 32
framebuffer_tag_end:
```

If the MULTIBOOT_TAG_TYPE_FRAMEBUFFER exists when enumerating the tags returned by GRUB, a multiboot_tag_framebuffer structure can be populated with a framebuffer's information.  

```
Code from src/kernel/main.c

    for (tag = (struct multiboot_tag *)(mbd + 8); tag->type != MULTIBOOT_TAG_TYPE_END; tag = (struct multiboot_tag *) ((multiboot_uint8_t *) tag + ((tag->size + 7) & ~7))) {
        switch (tag->type) {
        ...
            case MULTIBOOT_TAG_TYPE_FRAMEBUFFER:
                frametag = (struct multiboot_tag_framebuffer *)tag;
                privileged_addr[i++] = frametag->common.framebuffer_addr;
                break;
        ...
        }
    }
```

While working on the MemoryManagement module, we discovered that while GRUB does return memory regions marked as free, these free regions may contain reserved regions as well.    
For instance, it seems that the framebuffer is always placed in RAM blocks marked as free.     
The privileged_addr array seen in the code above is used to manually exclude these regions while GRUB tags are being enumerated.      

Notes:
- The pointer to the framebuffer returned by GRUB is always a contiguous range of memory.   
- The kernel itself has to map this region of memory.  
- One mustn't forget that page frames have to be mapped from the lowest address to the highest address.
- As processes and scheduling were implemented last, this PoC kernel maps the framebuffer into kernel space instead of a user process.

Assuming GRUB did return the framebuffer with the requested options, the framebuffer has a size of exactly 1920 * 1080 * 4 (32bit depth) bytes.     
Since the framebuffer has a depth of 32bit, 4 sequential bytes describe one pixel in the RGBA format.   
The first 4 bytes describe pixel 0 located in the top-left corner of the screen.    
Each subsequent pixel is displayed from left to right, top to bottom.

## Displaying text



## Displaying the cursor



## Displaying images (Unused)



## Command Handlers


