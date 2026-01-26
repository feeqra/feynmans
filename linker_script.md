Key:
UQUESTION = UNANSWERED QUESTION
AQUESTION = ANSWERED QUESTION


A linker script (ending in .ld) is essentially a map that tells the linker exactly how to
 organize your compiled code and data inside the microcontroller's memory.

 When you compile code, you get "object files" (.o) that contain machine code but don't know where
 in memory they will live. The linker script solves this by defining the physical memory addresses
 for every part of your program.

 Here is a breakdown of the key parts from your specific file (STM32L476RGTX_FLASH.ld):

 1. Memory Layout (MEMORY block)
 This defines the available hardware memory on your STM32L476 chip.

  1 MEMORY
  2 {
  3   RAM    (xrw)    : ORIGIN = 0x20000000,   LENGTH = 96K   /* Main volatile memory */
  4   RAM2    (xrw)   : ORIGIN = 0x10000000,   LENGTH = 32K   /* Extra RAM */
  5   FLASH    (rx)   : ORIGIN = 0x8000000,    LENGTH = 1024K /* Non-volatile code storage */
  6 }
  * FLASH: Where your code lives permanently (even when power is off).
  * RAM: Where variables live while the program is running.

 2. Grouping Data (SECTIONS block)
 This tells the linker which "sections" of code go into which memory area.

  * `.isr_vector`: The Interrupt Vector Table. This must be at the very beginning of FLASH so the
    processor knows where to start (Reset_Handler) and how to handle interrupts.
  * `.text`: Your actual program code (instructions) and constants (.rodata). This goes into
    FLASH.
  * `.data`: Global variables that have an initial value (e.g., int count = 5;).
      * Tricky part: These values must be saved in FLASH (so they survive power loss) but copied
        to RAM at startup so you can modify them. The script defines _sidata (source in flash) and
        _sdata/_edata (destination in RAM) to help the startup code perform this copy.
  * `.bss`: Global variables with no initial value (e.g., int counter;). The startup code uses the
    _sbss and _ebss symbols to zero out this area of RAM.

 3. Entry Point & Stack
  * `ENTRY(Reset_Handler)`: Tells the CPU, "When you wake up, go here first."
  * `_estack`: Calculates the top of the stack (End of RAM). The stack grows downwards from here.
  * `_Min_Heap_Size` / `_Min_Stack_Size`: Reserves safety margins to ensure your program doesn't
    run out of memory immediately.

 Why is this important?
 Without this file, the microcontroller wouldn't know that 0x8000000 is where instructions should
 be stored, or that 0x20000000 is writable memory. If you ever need to use that extra RAM2 or
 place a specific function at a fixed address (like for a bootloader), you would edit this file.


If your code is larger than the allocated FLASH memory defined in the linker script, the build
 process will fail. The linker will refuse to generate the final binary file.

 Here is exactly what happens:

  1. Linker Check: As the linker places your code (the .text and .rodata sections) into the FLASH
     region, it keeps a running count of how much space is used.
  2. Overflow Detection: It compares the total size against the LENGTH = 1024K defined in your
     MEMORY block.
  3. Error Message: If the size exceeds the limit, the linker stops and outputs an error similar
     to this:
  1     c:/.../ld.exe: region `FLASH' overflowed by 1234 bytes

 The Consequence:
 You will not get a .bin or .elf file to flash onto your board. You literally cannot load the
 program because it physically won't fit into the chip's storage.

 How to fix it:
 If you see this error, you usually have three options:
  1. Optimize: Turn on compiler optimizations (e.g., -Os for size) to make the code smaller.
  2. Cut Features: Remove unused libraries, debug strings, or complex logic.
  3. Upgrade: Switch to a microcontroller with more Flash memory.




AQUESTION: Can you explain more about why the isr vector table must be at the beginning of flash? Because the reset handler is itself an interrupt? Okay sure, but why can't the location be dynamic, and somewhere we store it's location. Surely that's possible but someone just made a design choice to always have it at the beginning?

You are hitting on a fundamental concept in computer architecture called the **Bootstrapping Paradox** (or the "Chicken and Egg" problem).

You are absolutely right that the location *can* be dynamic—and on modern microcontrollers, it often is. However, the **Reset Vector** specifically is the one exception that enforces the rule.

Here is the breakdown of why the hardware is designed this way.

### 1. The "Cold Start" Problem

When you first apply voltage to the microcontroller, the CPU is in a "dumb" state. It has:

* No operating system.
* No stack.
* No variables.
* **No knowledge of your software's layout.**

If the vector table location were dynamic (stored in a variable or a pointer), the CPU would need to know **where** to find that pointer.

If you said, *"Let's store the address of the vector table at address `0x2000`,"* then `0x2000` effectively becomes the new hard-coded fixed address. You cannot escape the need for at least **one** fixed, hardware-defined address to start the chain of execution.

### 2. The Hardware "Contract"

To solve the cold start problem, chip architects (like those at ARM) create a hardware contract. The silicon logic is hardwired to perform a specific sequence immediately upon power-up, usually something like:

1. **Fetch Initial Stack Pointer (MSP):** The hardware blindly reads the memory at address `0x00000000`.
2. **Fetch Reset Vector (PC):** The hardware blindly reads the memory at address `0x00000004`.
3. **Jump:** The Program Counter (PC) jumps to the address it just found.

Because the CPU blindly reads `0x0000` and `0x0004` before executing a single instruction, there is no opportunity for software to say, *"Actually, look over here instead."*

### 3. You *Can* Move It (VTOR)

Your intuition that "someone just made a design choice" is actually correct regarding what happens *after* boot.

On ARM Cortex-M processors (and many others), there is a register called **VTOR (Vector Table Offset Register)**.

* **At Boot:** VTOR defaults to `0` (the beginning of Flash).
* **In Software:** Your startup code (the Reset Handler) can copy the vector table to RAM, or point to a different section of Flash, and then update the VTOR with the new address.

Once you write to VTOR, the CPU will look at the new dynamic location for all *future* interrupts.

### Summary

The table must be at the beginning of Flash **only for the very first instruction**. The CPU needs a fixed "North Star" to know how to wake up. Once it is awake, you can move the table wherever you want.

UQUESTION: Who is responsible for keeping track of the max length of the ISR vector table? Like is it always the same size? Is it fixed size? So even if I don't create any more interrupts will that affect the final size of the vector table? My intuition tells me that we would not just have empty vector table entries so that we can save on precious space, so maybe somewhere else we store the variable length of the ISR vector table.

You are asking about the difference between a **Map** (key-value pairs, variable size) and an **Array** (fixed index, fixed offset).

Your intuition is optimizing for **space**.
The hardware is optimizing for **speed**.

Here is why we don't store the table as a variable-length list, and who is actually responsible for defining it.

### 1. The "Who": The Silicon Vendor & The Startup Code

There are two layers of responsibility here:

1. **The Silicon Vendor (e.g., ST, Nordic, TI):** When they design the chip, they hardwire the "slots." They decide that the "UART0 Interrupt" will **always** be at offset `0x80`. This is baked into the silicon.
2. **The Software (You/The Startup File):** The `startup_device.s` (assembly) or `.c` file defines the actual array in code. It lists the function pointers in the exact order the vendor specified.

### 2. Why it isn't Variable Length (The Math)

To support a variable-length table (where we delete unused entries to save space), the hardware would need to perform a search or a lookup.

* *Hardware would have to think:* "Okay, I need UART0. Let me check the table... is entry 1 UART0? No. Is entry 2 UART0? No..."

This is too slow (O(n) complexity). Interrupts need to be instant.

Instead, the hardware uses **O(1) Arithmetic**:


Because the hardware uses simple multiplication to find the address, **you cannot have gaps.** If the hardware expects the UART interrupt at Index 50, and you delete the unused Index 49 to save space, Index 50 shifts down to 49. When the UART fires, the hardware jumps to the wrong handler.

### 3. The "Swiss Cheese" Reality

Because of this math, the vector table is often full of "holes."

If your chip supports 100 interrupts, but you only use the very first one and the very last one, you must still allocate space for the 98 entries in between.

* **Q: What goes in the empty slots?**
* **A:** Usually a pointer to a `Default_Handler` (an infinite loop). If a "ghost interrupt" fires for some reason, the CPU jumps there and hangs, which is better than jumping into random code.

### 4. The One Optimization You *Can* Do

While you cannot delete entries in the *middle*, you technically can truncate the **end** of the table.

If the hardware supports 240 interrupts, but your application only uses the first 50, you *could* tell the Linker Script to stop the `.isr_vector` section after the 50th entry.

* **The Risk:** If for some reason interrupt #51 fires (maybe a glitch or bad configuration), the hardware will try to read memory *past* your table. It might read random garbage data as a function address, jump there, and crash.

### Summary

The vector table is a **Fixed-Index Array**, not a Linked List.

* **Who tracks the max length?** The hardware limits are in the datasheet (NVIC implementation). The actual length is defined in your **Linker Script** (`.ld` file) and **Startup Object**.
* **Is it fixed size?** It is fixed *width* (usually 4 bytes per entry). The total *length* is determined by the highest interrupt number supported by the chip.


UQUESTION: If the ISR vector table is a fixed size and is always placed at the start of flash memory, shouldn't we specify to flash our actual code at START_FLASH_MEM + LEN_ISR_VECTOR_TABLE?

UQUESTION: Constants go into flash?

UQUESTION: Why do global variables need to survive power loss? Are you saying we never want volatile global variables?

UQUESTION: If the ISR table always goes at the beginning of flash, I guess I find it redundant to specify in the linker file "put the ISR table at the beginning of flash". Just hardcode it to do that? Like is not the point of the linker file to give the user the ability to make modifications? So the fact that they put the ISR in the linker file communicates that there are some cases where you wouldn't want to put it in that hardcoded spot: beginning of flash?

UQUESTION: You said global variables with initial values are put into flash and RAM, but you didn't say where global variables with no initial values are placed.

UQUESTION: What the is .bss? What does it stand for and why is that an intuitive name?

UQUESTION: What happens when you max out ram then try to use more ram? like if you create a trillion global variables with initial values?
