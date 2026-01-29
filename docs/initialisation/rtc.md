# RTC Initialisation

RTC stands for Real Time Clock and its purpose is to keep the current date of the system even when the computer is in a shutdown state.     
The RTC is updated at a rate of 1 update per second.    
For this PoC OS, we decided to leverage it as a way to get accurate time as reported by the system.
N.B: According to OSDev, it is better to read it at boot time and then use another component to handle timers.

In 2009, Intel released a paper detailing the [RTC's register access](https://www.singlix.com/trdos/archive/pdf_archive/real-time-clock-nmi-enable-paper.pdf).     
Coupled with the [Linux kernel's documentation listing the contents of each RTC register](https://docs.kernel.org/virt/kvm/x86/timekeeping.html#rtc), we deduced the correct commands that the RTC must receive.

By writing value 0x8B to the index register 0x70:
* Interrupts remain disabled because bit 7's value is 1.
* RTC register B is selected.

Like the PIC, the index register has an associated Data register holding the data of the selected register.     
By reading 0x71, we retrieve the current state of interrupts configuration as well as the state of the clock's information such as 12-hour mode.
Shifting bit 4 to one and then writing the value into RTC register B will ensure that the RTC will fire an IRQ each time it was updated.    
The IRQ linked to the RTC is IRQ8 in a majority of cases according to the Linux Documentation.

## RTC Modes

The x86_64 RTC possesses 4 possible states when returning the current time:
* BCD Mode/12-hour Mode
* BCD Mode/24-hour Mode
* Binary Mode/12-hour Mode
* Binary Mode/24-hour Mode

N.B: One has to note that the mode the RTC is in may not be reprogrammable, meaning one must support all combinations if the RTC is handled by the OS.

### 12-hour/24-hour Mode

If bit 1 of register B is 0, the RTC is in 12-hour mode.    
If its value is 1, it is in 24-hour mode instead.   

In 12-hour mode, the hour register returns values between 1 and 12 included with midnight being 12AM.   
In 24-hour mode, midnight is 0.

### BCD/Binary Mode

If bit 2 of register B is 0, the RTC registers return values in BCD mode.   
If its value is 1, they return Binary values instead.

In Binary Mode, the values are interpreted as classic integers.     
For example, the time 11:47:33 (in 24-hour mode) is returned as 11 by the hour register, 47 by the minute register and 33 by the seconds register.  

In BCD Mode, the values are returned with a hexadecimal representation instead.     
For example, the time 11:47:33 (in 24-hour mode) is returned as 0x11 by the hour register, 0x47 by the minute register and 0x33 by the seconds register.
