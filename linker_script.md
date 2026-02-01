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

AQUESTION: Who is responsible for keeping track of the max length of the ISR vector table? Like is it always the same size? Is it fixed size? So even if I don't create any more interrupts will that affect the final size of the vector table? My intuition tells me that we would not just have empty vector table entries so that we can save on precious space, so maybe somewhere else we store the variable length of the ISR vector table.

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


AQUESTION: If the ISR vector table is a fixed size and is always placed at the start of flash memory, shouldn't we specify to flash our actual code at START_FLASH_MEM + LEN_ISR_VECTOR_TABLE?

Yes, you are 100% correct.

The code *must* start after the vector table. If you tried to write your `main()` function at address `0x00000000`, you would overwrite the Stack Pointer and Reset Vector, and the chip would brick immediately.

However, you rarely have to hard-code that offset math (e.g., `0x08000000 + 0x194`) yourself. The **Linker Script** handles the "cursor" for you.

Here is how the Linker Script automates this so you don't have to count bytes manually.

### 1. The "Cursor" Analogy

Imagine the Linker is a writer with a cursor.

1. You tell it: "Start at the beginning of Flash."
2. You tell it: "Write the Vector Table." -> The cursor moves down by the size of the table.
3. You tell it: "Write the Code (`.text`)." -> The writer starts writing code **exactly where the cursor stopped**.

### 2. The Linker Script (`.ld`)

Here is a simplified view of what happens inside the `SECTIONS` block of your linker script:

```ld
/* Define the start of Flash memory */
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K
}

SECTIONS
{
    /* 1. Place the Vector Table FIRST */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector)) /* Keep this section even if unused */
        . = ALIGN(4);
    } > FLASH

    /* The 'Cursor' is now at 0x08000000 + Length_of_Vectors */

    /* 2. Place the Program Code (.text) RIGHT AFTER */
    .text :
    {
        . = ALIGN(4);
        *(.text)           /* Your code (.c files) */
        *(.rodata)         /* Read-only constants */
        . = ALIGN(4);
    } > FLASH
}

```

* **`KEEP(*(.isr_vector))`**: This commands the linker to take the array defined in your startup file and stick it at the very top.
* **The Flow**: Because `.text` is listed *after* `.isr_vector`, the linker automatically calculates the end address of the vectors and starts placing your code immediately after it.

### 3. Verification ( The `.map` file)

If you ever doubt this is happening, you can open the **.map file** generated after you build. It explicitly lists where everything landed.

You will see something like:

| Section | Address | Size | Content |
| --- | --- | --- | --- |
| `.isr_vector` | `0x08000000` | `0x188` | (The Table) |
| `.text` | `0x08000188` | `0x2040` | `main`, `HAL_Init`, etc. |

Notice that `.text` starts exactly where `.isr_vector` ended (`0x08000000 + 0x188`).

### Summary

You don't typically calculate `START_FLASH_MEM + LEN_ISR_VECTOR_TABLE` manually. You simply list the Vector Table section **first** in the Linker Script, and the Linker pushes everything else down automatically.


AQUESTION: Why do global variables need to survive power loss? Are you saying we never want volatile global variables?

This is a crucial misunderstanding to clear up!

**Standard global variables DO NOT survive power loss.**

If you have a global variable `int counter = 0;`, and you increment it to `500`, then pull the plug and plug it back in... **`counter` will be `0` again.**

Here is the distinction between "Survival" (Flash) and "Operation" (RAM).

### 1. The confusing part: "The Initial Value"

You might think global variables survive because if you write `int myGlobal = 42;`, it is always `42` when the system starts.

* **The `42` survives:** The *number* 42 is stored in **Flash** (which is non-volatile/permanent).
* **The `myGlobal` dies:** The *variable* itself lives in **RAM**.

**At Startup (The Copy Trick):**
When your chip wakes up, the very first thing the startup code does is copy the `42` from Flash into the RAM address for `myGlobal`.

* If you change `myGlobal` to `99` while running, that `99` is in RAM.
* When power dies, the RAM (and the `99`) is wiped.
* When power returns, the startup code grabs the original `42` from Flash again.

### 2. Why we WANT variables in Volatile Memory (RAM)

You asked: *"Are you saying we never want volatile global variables?"*

We **absolutely** want them in volatile RAM. Here is why we don't put variables in Flash:

1. **Speed:** RAM is incredibly fast to read/write. Flash is slow to write.
2. **Wear:** Flash memory physically degrades. You can only write to a specific Flash sector ~10,000 to 100,000 times before it breaks. If you put a loop counter in Flash, you would burn out your chip in seconds. RAM has infinite write cycles.
3. **Hardware execution:** The CPU is designed to manipulate data in registers and RAM.

### 3. "The Exception": What if I *need* it to survive?

If you have data that truly needs to survive power loss (like "WiFi Credentials" or "Total Hours Run"), you cannot use a standard global variable.

You must specifically write code to "Save to Flash" or "Save to EEPROM."

1. **Read:** On boot, you read from that specific Flash address into a RAM variable.
2. **Write:** When the value changes, you trigger a Flash Erase/Program sequence (which takes time, sometimes milliseconds) to update the persistent storage.

### Summary

* **Vector Table:** MUST be in Flash (Non-Volatile) because the CPU needs it immediately upon wake-up, before it has time to "copy" anything.
* **Global Variables:** Live in RAM (Volatile). We want them there for speed and durability. Their *starting values* come from Flash, but their *current values* die with the power.


AQUESTION: If the ISR table always goes at the beginning of flash, I guess I find it redundant to specify in the linker file "put the ISR table at the beginning of flash". Just hardcode it to do that? Like is not the point of the linker file to give the user the ability to make modifications? So the fact that they put the ISR in the linker file communicates that there are some cases where you wouldn't want to put it in that hardcoded spot: beginning of flash?

This is a fantastic observation. You are essentially asking: *"Why do we need to manually configure something that seems to have only one valid option?"*

The answer lies in the fact that **"Beginning of Flash" is not the only place code can start.**

The Linker is a generic tool designed to handle complex memory maps. If the toolchain hardcoded the ISR table to the start of flash, it would break three major embedded workflows: **Bootloaders**, **RAM Debugging**, and **Dual-Bank Updates**.

Here is why that flexibility in the Linker Script is mandatory, not redundant.

### 1. The Bootloader Scenario (The #1 Reason)

In a professional product, you rarely have just "One Application." You usually have two programs on the chip:

1. **Bootloader (0x0800_0000):** Runs first. Checks if a firmware update is needed.
2. **Main App (e.g., 0x0800_4000):** Your actual product code.

If the linker *always* forced the ISR table to `0x0800_0000` (Start of Flash), you could **never** compile the Main App.

* The Main App needs its *own* vector table.
* That table must be placed at `0x0800_4000` (or wherever the App starts).
* By specifying this in the Linker Script, you generate a binary that is correctly offset to run from that higher address.

### 2. "Start of Flash" is Relative (Memory Aliasing)

On many ARM chips (like STM32), address `0x0000_0000` is actually an **alias**.

* Physical Flash might be at `0x0800_0000`.
* Physical RAM might be at `0x2000_0000`.
* The hardware maps `0x0000_0000` to point to *one* of these.

Sometimes, for debugging, you want to load your **entire program into RAM** (not Flash) to avoid wearing out the flash memory during rapid test cycles.

* In this case, you modify the Linker Script to put `.isr_vector` at `0x2000_0000` (Start of RAM).
* If the linker forced "Start of Flash," you physically couldn't run this debug configuration.

### 3. The "Generic Tool" Philosophy

The GNU Linker (`ld`) is used for everything from tiny 8-bit microcontrollers to massive Linux kernels running on supercomputers.

* It doesn't "know" you are using a Cortex-M4 that expects a table at reset.
* It only knows: *"You gave me bytes. You gave me addresses. I will put the bytes at the addresses."*

It leaves the architectural constraints (like "The hardware requires a table here") up to the **Linker Script author** (usually the silicon vendor who provides the template).

### Summary

The configuration seems redundant for a simple "Hello World" project, but it is critical for **Systems Architecture**.

* **Bootloaders:** Need vector tables in the *middle* of flash.
* **RAM Execution:** Needs vector tables in *RAM*.
* **Multi-Core:** Core A might boot from `0x0800_0000`, while Core B boots from `0x0810_0000`.

The linker script gives you the power to define "Beginning" however you want.

AQUESTION: So it would be more appropriate to say "you must put the isr vector table at the start of the application code."? Because the reset handler is the first entry of the ISR vector table?

You are getting closer to the perfect definition.

You are **95% correct**, but that missing 5% is the difference between a working system and a "Hard Fault" crash.

### 1. The Correction: It is NOT the First Entry

You asked: *"Because the reset handler is the first entry of the ISR vector table?"*

**No.** The Reset Handler is the **second** entry.

The Cortex-M hardware expects the very first 32 bits (4 bytes) to be the **Initial Stack Pointer (MSP)** value.

* **Address `0x00`:** Initial Stack Pointer Value (Where the stack starts in RAM).
* **Address `0x04`:** Reset Handler Address (Where the code starts in Flash).

If you put your Reset Handler (code instructions) at the very start (`0x00`), the CPU will try to load your machine code instructions as a memory address for the Stack Pointer. Your stack will point to a random location, and your chip will crash immediately.

### 2. The Better Definition: "The Application Header"

Your refined statement is excellent: **"The ISR vector table must be at the start of the application binary."**

Think of the Vector Table as the **File Header** for your firmware.

When a Bootloader wants to run your application, it doesn't just "jump to the start." It performs a handshake that relies on the table being at the start:

1. **Read App Base + 0:** "Okay, here is where this app wants its **Stack** to be." -> *Update SP register.*
2. **Read App Base + 4:** "Okay, here is where this app's **Code** begins." -> *Load into PC (Program Counter).*
3. **Write App Base to VTOR:** "Okay, all future interrupts for this app are located here."

### 3. Why "Start of Application" is the correct mental model

You are right to move away from "Start of Flash."

* **Bare Metal (No OS):** The "Application" starts at the "Start of Flash." (The Vector Table is at `0x08000000`).
* **With Bootloader:** The "Application" starts at `0x08004000` (or wherever you put it). The Vector Table must be at `0x08004000`.

In both cases, the **Vector Table anchors the application**. It defines the "Zero Point" for that specific piece of software.

### Summary

* **Correction:** The Reset Handler is the **2nd** entry. The **1st** is the Stack Pointer.
* **Refinement:** Yes, the Vector Table must be at the **Start of the Application Image**. It serves as the "Table of Contents" that tells the CPU (or bootloader) how to set up the environment before running the first line of C code.

---

**Would you like to see the specific 3 lines of C code a Bootloader uses to read that table and jump to the application?**

Here is the standard "Jump to Application" logic used in nearly every ARM Cortex-M bootloader.

This code assumes your application is flashed at `0x08004000`.

```c
#define APP_ADDRESS  0x08004000

void JumpToApplication(void)
{
    // 1. Read the Initial Stack Pointer (MSP) from Offset 0
    // We treat the address as a pointer to a uint32_t, then dereference it.
    uint32_t app_stack = *(volatile uint32_t*)APP_ADDRESS;

    // 2. Read the Reset Handler Address from Offset 4
    // We move the pointer 4 bytes forward, then dereference.
    uint32_t app_reset_handler_address = *(volatile uint32_t*)(APP_ADDRESS + 4);

    // 3. Set the MSP and Jump
    // We cast the address to a function pointer so we can "call" it.
    void (*app_reset_handler)(void) = (void*)app_reset_handler_address;

    __set_MSP(app_stack);         // Hardware intrinsic to set the Main Stack Pointer
    app_reset_handler();          // The actual jump!
}

```

### Why this confirms your "Application Header" theory:

Notice that the bootloader knows **nothing** about the application's code. It doesn't know where `main()` is. It doesn't know how big the app is.

It only trusts the **Vector Table contract**:

1. "I trust that the first 4 bytes at `APP_ADDRESS` are a valid Stack Pointer."
2. "I trust that the next 4 bytes are the start of your code."

If you compiled your application but forgot to put the vector table at the start, the bootloader would read random instruction opcodes (garbage) as the stack pointer and crash instantly.

### One Critical Detail (The "Gotcha")

There is one line of code missing above that you must add if your Bootloader used any interrupts.

Before you jump, you **must disable all interrupts** (`__disable_irq()`) and clear any pending interrupt bits.

* **Why?** If a timer interrupt fires *milliseconds* after you jump, the CPU will try to look up the handler.
* But wait! The **VTOR** (Vector Table Offset Register) is likely still pointing to the *Bootloader's* table.
* The CPU will try to run the Bootloader's Timer Handler, but the Bootloader's variables might not exist anymore. **Crash.**

---

**Would you like to see the Linker Script command (`MEMORY` command) that ensures the App creates this binary structure at the correct address?**

CQUESTION: Funny question, but like where is the default bootloader? Like where can I see that code you're saying is "in every cortex-M application" lol? I'm using STM nucleo L476RG. And like theoretically if I was making my own bootloader, do I just erase the existing "bootloader.c (example name)" file and replace it with my own code? Or like how does that work? Maybe there's a box "generate bootloader code" that you uncheck when generating your code?

This is the funniest and most common realization in embedded engineering: **"Wait, there is no file?"**

You are looking for a file named `bootloader.c`, but in your current project, **it does not exist.**

Right now, on your Nucleo L476RG, your "Blinky LED" program **IS** the Bootloader. It is the first thing the CPU runs.

Here is the breakdown of the "Default" vs. "Custom" confusion.

### 1. The "Default" Bootloader (The Hardware one)

There *is* a piece of code that was on your chip before you even bought it. It is called the **System Memory Bootloader**.

* **Where is it?** It is burned into a hidden "ROM" region of the chip (System Memory) by ST at the factory.
* **Can I see the code?** No. It is proprietary ST code.
* **Can I delete/replace it?** No. It is read-only.
* **What does it do?** It checks the **BOOT0 pin**. If you hold that pin high while resetting, this code runs instead of yours. It allows you to flash the chip via USB or UART if you accidentally "brick" it.

### 2. The "Startup" Code (The file you DO have)

When you said *"that code you said is in every application,"* you are likely looking at the **Startup File**.

* **File Name:** Usually `startup_stm32l476xx.s` (Assembly) in your generic `Core/Startup` folder.
* **What it does:** This is the *real* "default bootloader" of your application.
1. Initializes the Stack Pointer.
2. Copies variables from Flash to RAM.
3. Jumps to `main()`.



**You do not delete this.** Every program needs this to wake up.

### 3. "How do I make my own Bootloader?"

This is where your mental model of "unchecking a box" needs a slight shift.

You don't *modify* your current project to become a bootloader. You create **Two Separate Projects**.

**Project A: The Custom Bootloader**

* **Location:** You tell the Linker: "Put this at `0x08000000` (Start of Flash)."
* **Code:** Inside `main()`, it checks "Is the button pressed?"
* *Yes:* Enter "Update Mode" (wait for UART data).
* *No:* Run that `JumpToApplication()` function I showed you earlier.



**Project B: Your Actual Application**

* **Location:** You tell the Linker: "Put this at `0x08004000` (Start + Offset)."
* **Code:** This is your normal code.
* **The Change:** You *keep* the startup file (`startup_stm32...s`) in this project too! The application needs its own startup sequence to initialize its own variables after the jump.

### Summary

1. **Where is the code now?** The "Jump" logic I showed you **is not on your board yet**. It only exists if you write it.
2. **Do I erase `bootloader.c`?** No. You create a **new project** that lives at Address 0, and you move your **old project** to Address 1.
3. **The Box to Uncheck:** There is no box. The "switch" is purely inside the **Linker Script (`.ld`)**. You manually change `FLASH ORIGIN = 0x08000000` to `0x08004000` for the second project.

**Would you like to see the one-line change in the Linker Script that turns a "Normal App" into an "App that sits at 0x4000"?**

UQUESTION: > uint32_t app_stack = *(volatile uint32_t*)APP_ADDRESS; // So what is the data type of APP_ADDRESS before you cast it to a memory address? Like is it (and all other macros defined as 0x1234...) just 32 bit integers? What if you said #define example_macro 0x123456789123456789, like that's too big for 32 bits so like where is that stored?

UQUESTION: >     uint32_t app_stack = *(volatile uint32_t*)APP_ADDRESS; // I just don't understand. Why in the world are we immediately dereferencing a memory address that we just casted to a memory address? My understanding of this line of code is "cast a uint32 to a volatile pointer then cast it to a uint32"???

UQUESTION: > If you put your Reset Handler (code instructions) at the very start (`0x00`), the CPU will try to load your machine code instructions as a memory address for the Stack Pointer. Your stack will point to a random location, and your chip will crash immediately. // I think you meant to say "CPU will try to load the memory address of your machine code instructions"? Like it won't load the instruction itself just the address of the first location?

UQUESTION: > __set_MSP(app_stack); // I've never heard the words "hardware intrincsic" in my life.

UQUESTION: > * **Address `0x00`:** Initial Stack Pointer Value (Where the stack starts in RAM). But isn't that just "the start of RAM"? Like does not the stack always start at the beginning of RAM? If not I mean what is the STRUCTURE OF RAM? Isn't RAM just the stack and the heap?

UQUESTION: > **Write App Base to VTOR:** "Okay, all future interrupts for this app are located here." Gonna have to explain this to me? What are you saying? The app moves the ISR table to somewhere else? RAM? Because reading pointers from RAM is faster than re-reading it from flash?

UQUESTION: I don't understand why the application would crash if you put garbage as the first entry on the ISR vector table.

UQUESTION: Casting an address to a function pointer? This is literally voodoo magic. You've gotta explain this syntax to me:     void (*app_reset_handler)(void) = (void*)app_reset_handler_address;

UQUESTION: So wait theres an initial stack pointer and a main stack pointer? Two stack pointers? You've got to be kidding me.

UQUESTION: What does "volatile" do here:     uint32_t app_stack = *(volatile uint32_t*)APP_ADDRESS; // Like what would happen if we didn't put volatile?

UQUESTION: Wait are you saying the OS is written to flash? You differentiated "No OS" vs "with a bootloader". Are you saying a boot loader is an OS?

UQUESTION: You said global variables with initial values are put into flash and RAM, but you didn't say where global variables with no initial values are placed.

UQUESTION: What the is .bss? What does it stand for and why is that an intuitive name?

UQUESTION: What happens when you max out ram then try to use more ram? like if you create a trillion global variables with initial values?

UQUESTION: What are interrupt bits? Maybe I can see them in action right now in my STM nucleo L476RG?

UQUESTION: > Before you jump, you **must disable all interrupts** (`__disable_irq()`) and clear any pending interrupt bits. // So that implies the first thing main() must do is re-enable interrupts. And if a critical  life-saving interrupt gets called while we have interrupts disabled and someone dies (example) ????
