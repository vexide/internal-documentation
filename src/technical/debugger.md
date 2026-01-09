# VEX V5 Debugging

Despite [long-term interest](https://www.vexforum.com/t/debugger-for-vex-v5/91619) by the VEX community, there is currently no complete on-device debugger for the VEX V5 platform.

However, there has been some past work on implementing one on previous VEX platforms. For instance, the ROBOTC language and framework (which targeted the VEX Cortex platform) included a feature-rich [graphical debugger](https://www.robotc.net/WebHelpVEX/index.htm#Resources/topics/ROBOTC_Debugger/Program_Debugging.htm) that took advantage of the language's virtual machine environment. Additionally, the PROS project, also targeting the Cortex, initially included a prototype of a real-time, key/value dashboard called "JINX Debugger" (despite not including the traditional features of a [debugger](https://en.wikipedia.org/wiki/Debugger)). Ultimately, neither ROBOTC nor JINX were ported to the more recent VEX V5.

Notably, another more recent project is Interfiber's [v5dbg](https://github.com/Interfiber/v5dbg), a debugger for PROS programs that allows developers to define breakpoints at compile time using some C++ macro magic and function calls. It also allows users to manually expose variables so they can be viewed when their breakpoints are hit. Unfortunately, due to v5dbg's architecture, users cannot step through lines of code or place arbitrary breakpoints at runtime. Additionally, developers must update their existing code to take advantage of the debugger's features.

## The vexide debugger

In order to bring a full debugger to the VEX V5 platform, [we are working on implementing](https://github.com/vexide/vexide/tree/feat/v5-debugger/packages/v5-debugger) a GDB backend for VEX V5. This library takes advantage of the Cortex A9's advanced hardware debug functionality, adding the ability to place breakpoints in arbitrary locations and view variables without explicit code changes. This section gives a high-level overview of the architecture of this project.

While we are prioritizing ease of use in vexide, we plan to bring the vexide debugger to the PROS and VEXcode frameworks in the future.

### Resources

Those interested in contributing are highly encouraged to download a copy of the [ARMv7-A Reference Manual](https://developer.arm.com/documentation/ddi0406/latest) and optionally the [Cortex-A9 Manual](https://documentation-service.arm.com/static/5e8e2ab9fd977155116a7035?token=). These cover some important details about debug events, debug registers, and the CPU's exception model. Also, it might be helpful to look at the [Zynq 7000 Reference](https://docs.amd.com/viewer/book-attachment/mxcNFn1EFZjLI1eShoEn5w/~F6mCjSrb3_BDetD7782sA-mxcNFn1EFZjLI1eShoEn5w) for information about the device's memory map and configuration registers.

### ARMv7-A debugging model

The VEX V5 is built around the Zynq 7000 system-on-chip, running user code on one of its dual ARM Cortex-A9 processors. Since all user code is run in System mode (aka ring 0 or kernel mode), it is free to manage processor state such as the debug architecture. The debugger library uses this ability to configure the CPU just as a real operating system would.

The ARMv7-A debug architecture is controlled by various registers that the CPU checks to determine where, when, and how to cause breakpoints. These registers can be accessed through [Coprocessor #14](https://developer.arm.com/documentation/den0013/0400/ARM-Architecture-and-Processors/Architecture-history-and-extensions/Coprocessors) (cp14), but require special assembly instructions for each one - accessing each numbered hardware breakpoint would have to be its own line of code, for instance. The alternative is the memory-mapped I/O interface, which lets you access processor state like any other global integer or array. For the most part, this is what the vexide debugger uses to access hardware state: it reads and saves the address of the MMIO interface from cp14 when it initializes.

The debugging process itself is centered around *debug events*, which can occur whenever a breakpoint, watchpoint, or `bkpt` assembly instruction are triggered. Depending on the CPU state, debug events will either be ignored (the default), cause the CPU to halt (for debugging with a probe device), or induce a debug exception that can be handled by the operating system. The V5's array of hardware breakpoints is limited, so, for the most part, we just dynamically edit programs to replace certain instructions with `bkpt`. Whenever they're hit, a debug event is triggered just as if a real breakpoint had been there. When we want to continue, we can replace the `bkpt` instruction with whatever was there before and return from the exception handler. This approach is called [software breakpoints](https://en.wikipedia.org/wiki/Breakpoint#Software) because it doesn't require the use of the hardware's dedicated breakpoint registers.

Hardware breakpoints do have their uses, though, because they can be configured to trigger when the processor _isn't_ executing a targeted instruction rather than just when it is doing so. For instance, if the debugger wanted to single-step past an instruction, it could place such a "mismatch" breakpoint on the current instruction and return from the debug handler. Then, just as that instruction finished, the program would pause again. Recreating this same behavior would be pretty difficult with just the strategy of overwriting certain instructions with `bkpt` because programs don't always advance linearly: they might call a function, for instance, or jump back to the beginning of a loop. Using a hardware breakpoint to avoid having to guess what the next instruction will be helps the debugger avoid implementing what'd essentially be an entire ARM emulator.

(See also chapters C1 and C3 of the ARMv7-A manual.)

### Zynq 7000 configuration

Unfortunately, using hardware breakpoints requires some more set up because they only cause debug events if the Cortex-A9 processor's DBGEN (Debug Enable) pin is set high. In comparison, the `bkpt` instruction always causes a debug event (and, in turn, those events always cause exceptions the library is able to handle). This requires the debugger to change the configuration of the Device Configuration Interface, one of the peripherals on the Zynq itself, to take advantage of the flexibility of hardware breakpoints.

By default, VEXos configures the Memory Management Unit to protect that peripheral against accidental reads and writes by the user code CPU. Since the vexide debugger is accessing it legitimately, it temporarily disables the MMU in a [critical section](https://en.wikipedia.org/wiki/Critical_section) when it needs to update the configuration. Note that it is unlikely VEXos resets either the Zynq's device configuration or CPU's breakpoint configuration after each program, so it's likely that breakpoints persist between different programs (but not between restarts). This is not a problem that has been investigated yet.

(DBGEN is discussed in chapter C2 of the ARM manual.)

### CPU exceptions and the vector table

As mentioned previously in this document, the vexide debugger is able to capture the emitted debug events, execute some code in a debug monitor state, then continue execution where it left off. It uses the processor's CPU exceptions feature to facilitate this. Exceptions are used for anything from interrupts (common) to [data/prefetch aborts](https://en.wikipedia.org/wiki/Segmentation_fault) (hopefully never). Whenever an exception occurs, the processor saves its current state and re-starts execution at a predictable but varying offset inside the currently-configured _exception vector table_. Traditionally, this is a list of jump instructions to functions that know how to handle whatever the current "exceptional" process state is. For instance, after a prefetch abort, the processor jumps to `vector_table + 0x0c`. There, vexide's vector table has a second jump instruction to its code for showing an abort error screen.

![A diagram of vexide's vector table showing that some exceptions are handled by the library itself and others are handled by the VEX SDK.](./vector-table-simple.excalidraw.svg)

The vexide debugger needs its own vector table to manage debug exceptions triggered by breakpoints. However, since a processor can only have one vector table active at a time, any vector table we install must also be able to handle the responsibilities of any exception handlers configured by the user’s framework (e.g., vexide or PROS). Instead of incorporating framework-specific logic to simply redirect to those specific handlers, we opted to have our vector table function as a flexible overlay. When the debugger is initialized, it stores the items in the current vector table, anticipating the presence of an existing set of handlers that are already set up and capable of handling the standard exceptions. It then re-configures the processor to use its own vector table, which is specially configured to fall through for any exceptions out of its scope.

![A diagram of the debugger's overlay vector table showing that some exceptions are handled by the debugger and others fall through to the framework's original set of exception handlers.](vector-table-overlay.excalidraw.svg)

This architecture is useful because it makes few assumptions. For instance, if another error-handling library overwrites the vector table with its own custom data abort handlers, the debugger will be completely compatible with it at runtime because its own vector table will dynamically fall through to whatever else is currently in use.

### Persistent breakpoints

When a debugger wants to continue execution after a breakpoint, it initially seems reasonable to just try returning from that breakpoint's exception handler. However, it wouldn't have much luck doing so without changing some state first: if the program tried to return back to its old state, the exact same breakpoint would just fire again! This would leave the user in an infinite loop, unable to continue until they decide to disable the breakpoint that originally caused their program to halt. It turns out, continuing program execution just isn't possible without temporarily disabling this breakpoint.

To solve this problem, the vexide debugger single-steps one instruction with the original breakpoint turned off. Then, since the processor is no longer affected by the original breakpoint, the debugger turns it back on and continues execution as normal. As such, one hardware breakpoint must be reserved for single stepping even if the user isn't using that functionality directly.
