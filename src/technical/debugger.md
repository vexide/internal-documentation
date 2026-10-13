# VEX V5 Debugger (v5gdb) Internals

The vexide team has developed v5gdb, an [on-device debugger](https://github.com/vexide/v5gdb) for the VEX V5 platform to satisfy a [long-term interest](https://www.vexforum.com/t/debugger-for-vex-v5/91619) by the VEX community for better debugging tools. This chapter will attempt to give an overview of how v5gdb works and the problems we needed to solve to create it.

## Previous Work

There has been some past work on implementing a similar project on previous VEX platforms. For instance, the ROBOTC language and framework (which targeted the VEX Cortex platform) included a feature-rich [graphical debugger](https://www.robotc.net/WebHelpVEX/index.htm#Resources/topics/ROBOTC_Debugger/Program_Debugging.htm) that took advantage of the language's virtual machine environment. Additionally, the PROS project, also targeting the Cortex, initially included a prototype of a real-time, key/value dashboard called "JINX Debugger" (however, it never was expanded to include most of the traditional features of a [debugger](https://en.wikipedia.org/wiki/Debugger)). Ultimately, neither ROBOTC nor JINX were ported to the more recent VEX V5.

Another more recent notable project is Interfiber's [v5dbg](https://github.com/Interfiber/v5dbg), a debugger for PROS programs that allows developers to define breakpoints at compile time using some C++ macro magic and function calls. It also allows users to manually expose variables so they can be viewed when their breakpoints are hit. Unfortunately, due to v5dbg's architecture, users cannot step through lines of code or place arbitrary breakpoints at runtime. Additionally, developers must update their existing code to take advantage of the debugger's features.

With v5gdb, our goal was to bring a "full" debugger on par with what ROBOTC had supported to the VEX V5 platform, so we decided to instead implement a GDB backend for VEX V5. By taking advantage of the Cortex A9's advanced hardware debug functionality, we were able to implement the ability to place breakpoints in arbitrary locations and view variables without explicit code changes. While this debugger's primary goal is to support vexide and PROS, we would like to bring support to the VEXcode framework in the future.

## Debugger overview

This section gives a high-level overview of the architecture of this project.

### Resources

Those interested in contributing are highly encouraged to download a copy of the [ARMv7-A Reference Manual](https://developer.arm.com/documentation/ddi0406/latest) and optionally the [Cortex-A9 Manual](https://documentation-service.arm.com/static/5e8e2ab9fd977155116a7035?token=). These cover nearly all important details about debug events, debug registers, and the CPU's exception model. Also, it might be helpful to look at the [Zynq 7000 Reference](https://docs.amd.com/viewer/book-attachment/mxcNFn1EFZjLI1eShoEn5w/~F6mCjSrb3_BDetD7782sA-mxcNFn1EFZjLI1eShoEn5w) for information about the device's memory map and configuration registers.

While v5gdb interacts with VEXos fairly minimally, it may also be helpful to refer to some of the other resources on this website that cover how VEXos itself works.

### ARMv7-A debugging model

The VEX V5 is built around the Zynq 7000 system-on-chip, running user code on one of its dual ARM Cortex-A9 processors. Since all user code is run in System mode (which is one of the privileged "kernel modes"), it is free to manage processor state such as the debug architecture. The debugger library uses this ability to configure the CPU just as a real operating system would.

The ARMv7-A debug architecture is controlled by various registers that the CPU checks to determine where, when, and how to cause breakpoints. These registers can be accessed through [Coprocessor #14](https://developer.arm.com/documentation/den0013/0400/ARM-Architecture-and-Processors/Architecture-history-and-extensions/Coprocessors) (cp14), but require special assembly instructions for each one - for instance, accessing each numbered hardware breakpoint would have to be its own line of code. The alternative is the processor's debug MMIO interface, which lets you access debug state just like any other global integer or array. For the most part, v5gdb uses MMIO to access hardware state because of its greater ease-of-use. To access it, the library reads and saves the address of the MMIO interface from cp14 when it initializes.

The debugging process itself is centered around *debug events*, which can occur whenever a hardware breakpoint, watchpoint, or `bkpt` assembly instruction are triggered. Depending on the CPU state, debug events will either be ignored (the default), cause the CPU to halt (for debugging with a probe device like J-Link), or induce a debug exception that can be handled by the operating system. The V5 only supports so many hardware breakpoints, so, for the most part, we just dynamically edit programs to replace certain instructions with `bkpt`. Whenever they're hit, a debug event is triggered just as if a real breakpoint had been there. When we want to continue, we can replace the `bkpt` instruction with whatever was there before and return from the exception handler. This approach is called [software breakpoints](https://en.wikipedia.org/wiki/Breakpoint#Software) because it doesn't require the use of the hardware's dedicated breakpoint registers.

Hardware breakpoints do have their uses, though, because they can be configured to trigger when the processor _isn't_ executing a targeted instruction rather than just when it is doing so. For instance, if the debugger wanted to single-step past an instruction, it could place such a "mismatch" breakpoint on the current instruction and return from the debug handler. Then, just as that instruction finished, the program would pause again. Recreating this same behavior would be pretty difficult with the normal strategy of overwriting certain instructions with `bkpt` because programs don't always advance linearly: they might call a function, for instance, or jump back to the beginning of a loop. Using a hardware breakpoint to avoid having to guess what the next instruction will be helps the debugger avoid implementing what'd essentially be an entire ARM emulator.

(See also chapters C1 and C3 of the ARMv7-A reference.)

### Zynq 7000 configuration

Unfortunately, using hardware breakpoints at all requires some more setup because they only work if the Cortex-A9 processor's DBGEN (Debug Enable) signal is set High. In comparison, the `bkpt` instruction always causes a debug event (and, in turn, `bkpt` debug events always cause exceptions the library is able to handle). Setting DBGEN requires the debugger to access the Device Configuration Interface, one of the peripherals on the Zynq itself. Without doing this, it would be unable to take advantage of the flexibility of hardware breakpoints.

By default, VEXos configures the Memory Management Unit (MMU) to protect that peripheral against accidental reads and writes by the user code CPU. Since the vexide debugger is accessing it legitimately, it temporarily disables MMU permission checks by switching to the Manager Domain when it needs to update the device configuration. Note that it is unlikely VEXos resets either the Zynq's device configuration or CPU's breakpoint configuration after each program, so it's likely that breakpoints persist between different programs (but not between restarts). This is not a problem that has been investigated yet.

(DBGEN is discussed in chapter C2 of the ARMv7-A reference.)

### CPU exceptions and the vector table

As mentioned previously in this document, the vexide debugger is able to capture the emitted debug events, execute some code in a debug monitor state, then continue execution where it left off. It uses the processor's CPU exceptions feature to facilitate this. Exceptions are the system the CPU uses to handle anything that requires it to pause and switch context, from interrupts (common) to [data/prefetch aborts](https://en.wikipedia.org/wiki/Segmentation_fault) (hopefully never). Whenever an exception occurs, the processor saves its current state and re-starts execution at one of several offsets inside the currently-configured _exception vector table_. Traditionally, this is a list of jump instructions to functions that know how to handle whatever the current "exceptional" process state is. For instance, after a prefetch abort, the processor jumps to `vector_table + 0x0c`. There, vexide's vector table has a jump instruction to its code for showing an abort error screen.

![A diagram of vexide's vector table showing that some exceptions are handled by the library itself and others are handled by the VEX SDK.](./vector-table-simple.excalidraw.svg)

The vexide debugger needs its own vector table to manage debug exceptions triggered by breakpoints. However, since a processor can only have one vector table active at a time, any vector table we install must also be able to handle the responsibilities of any exception handlers configured by the user’s framework (e.g., vexide or PROS). Instead of incorporating framework-specific logic in the vector table, we opted to have ours function as a flexible overlay. When the debugger is initialized, it stores the items in the original vector table, assuming that it contains an existing set of handlers that are already capable of handling the normal exceptions. It then re-configures the processor to use its own vector table, which captures anything debug-related but falls through for any exceptions out of its scope.

![A diagram of the debugger's overlay vector table showing that some exceptions are handled by the debugger and others fall through to the framework's original set of exception handlers.](vector-table-overlay.excalidraw.svg)

This architecture is useful because it makes fewer assumptions. For instance, if another error-handling library adds its own custom abort handlers to the exception vector table, the debugger will be completely compatible with it at runtime because its own overlay vector table will dynamically fall through to whatever else is currently in use.

### Interrupt Handlers

When debugging a program with GDB, you can press Ctrl-C to send the device a packet asking to pause the program. In order to periodically check for these Ctrl-C packets, v5gdb runs some of its own code before each IRQ exception to poll serial input. Since IRQ exceptions happen fairly regularly (e.g. the CPU1 Timer triggers one every 1ms) and the serial input buffer is filled automatically in the background by CPU0, this approach allows the program to detect when GDB has asked it to pause even when the user's code has otherwise frozen up.


In some cases, instead of just running some code before the user's normal IRQ handler, v5gdb will completely take over servicing interrupts. Especially under FreeRTOS, the normal system interrupt handlers tend to edit state or switch CPU modes in a way that is incompatible with the debugger. This was the cause of some trouble in the past: at one point, if an interrupt was triggered while the debugger was arming a single step, that single step would fire inside the framework interrupt handler instead of in the user's code. That would lead to the debug monitor starting recursively, which led to all sorts of other trouble.

## Implementation

### GDB protocol and transports

The `gdb` command line tool is able to connect to remote devices (such as the VEX V5) and issue them commands like "create a breakpoint at \<ADDRESS\>" or  "send the values of all general-purpose registers." From the user's perspective, the experience is as seamless as on-device debugging, but it does require GDB to reliably and and receive packets and be able to recognize what data from the remote device is intended for it (versus what data is just noise). To facilitate the flexible streaming of this data between the computer running GDB and the robot hosting the debugger, vexide's debugger allows developers to implement custom _transport streams_.

The simplest possible transport stream is just sending GDB's packets as-is across the V5's USB serial connection. This was the original communication method used by the debugger. With this approach, a user might connect to their brain by telling GDB to pipe its packets directly through cargo-v5's terminal connection:

```
target extended-remote | cargo v5 terminal
```

While this approach is very convenient, it has the notable downside of preventing developers from using the V5's serial stream for _any other purpose_ such as logging or tracing. Thus, a more flexible approach is to add extra packet framing to all data leaving the device which allows data to be distinguished between coming from the user or coming from v5gdb. Then, a program running on the user's computer decodes it. This is what v5gdb calls `StdioTransport`.

### Cache coherency for self-modifying code

Software breakpoints are implemented by temporarily replacing certain instructions in a program with `bkpt` instructions. This is considered a type of [self-modifying code](https://en.wikipedia.org/wiki/Self-modifying_code) similar to JIT compilers or vexide's differential patcher, and as such, is subject to some of the same caching problems. Modern processors like the V5's Cortex-A9 have both instruction caches and data caches to speed up program execution, but they work under the assumption that "instructions" and "mutable data" are separate. As such, after we insert or remove a breakpoint, we need to take care to correctly manage both of these caches to ensure the change actually takes effect.

When the breakpoint instruction is initially changed in memory, it likely isn't actually written to RAM at first; instead, the change is kept track of by the processor's internal data cache. As it turns out, the part of the processor responsible for managing memory is separate from the part that fetches instructions. For any new breakpoint to take effect, these caches must be synchronized to the [Level of Unification](https://wiki.osdev.org/ARMv7_Common_Memory_System_Architecture#Level_of_Unification_%28verbatim%29), the point at which all parts of the processor see the same copy of the system memory. The debugger does this by first _cleaning_ the 32-byte section of the data cache containing the modified memory (committing it to system memory), then _invalidating_ the same part of the instruction cache (pulling the new changes from system memory). After this is completed, it's also best practice to invalidate the processor's branch prediction system to help prevent any slow state rollbacks in case of a misprediction.

(The cache operations required after self-modifying code operations are described in the ARMv7-A reference in section D7.5.2 "Instruction cache maintenance operations".)

### Debug register access

The Cortex-A9's ARMv7-A architecture defines a plethora of debug registers that can be used to control the processor's breakpoints and watchpoints, among other things. There are two methods for accessing them: either as registers in Coprocessor 14 (cp14) or as a `struct` at a special place in memory. The latter is preferable because it works better with Rust's ownership model and is overall simpler to use. While each access to cp14 requires a specially encoded `mcr` or `mrc` assembly instruction, it's trivial to index an array in memory. However, in order to take advantage of struct-like MMIO method, the debugger makes a few accesses to cp14 when it first initializes to determine the base address of those memory-mapped registers. This is done by accessing two registers:

- `DBGDRAR` (Debug ROM Address Register) defines the address in memory of a ROM Table that encodes where to access the debug registers of each processor on the device. We don't access this table whatsoever.
- `DBGDSAR` (Debug Self Address Offset Register) defines the offset of the current processor's debug registers in memory from the start of the ROM Table.

Adding these two values together gives a pointer to the virtual struct containing debug registers.

(Chapter C6 of the ARMv7-A manual describes the cp14 interface and the memory-mapped struct interface. Chapter C11 has a summary of the various registers that the debugger uses.)

### Exception handling

When breakpoints are hit, they generally turn into prefetch aborts. In the debugger's handler for this kind of exception, it determines whether the abort is "real" or not by checking if the "cause" bits in the Instruction Fault Status Register indicate a debug event. If so, execution diverges to the debugger. Importantly, throughout this process, no general-purpose registers are clobbered.

Once the debugger knows it's going to stop, it collects the current state of the processor for inspection and modification by the GDB client inside an instance of `DebugEventContext`. Then, when the debugger is ready to return, it restores the (possibly modified) general purpose registers and processor state before jumping back to user code.

Support for dedicated floating point registers isn't implemented right now, which means it's not possible to see some floating point values from GDB. The main work for this feature will be updating the exception handling code to store these in the context struct.

### User setup and usage

Currently, users activate the debugger by passing a transport method to the `install` function. They then enter debug mode by explicitly calling the `breakpoint` function.

```rust
#[vexide::main(banner(enabled = false))]
async fn main(_peripherals: Peripherals) {
    v5gdb::install(StdioTransport::new());

    v5gdb::breakpoint();

    let iters = black_box(80);
    let n = fib(iters);
    black_box(n);
}
```

This process may need to be refined before the debugger is finished.
