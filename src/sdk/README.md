# VEX SDK Tech Notes

The VEX SDK is a closed-source library of functions built-in to VEXos which can be called by user code to perform I/O operations like moving motors, reading sensors, and controlling the V5 brain's builtin LCD screen. Since the SDK is largely undocumented, it can be difficult to develop software (like vexide) which requires its functionality. This section includes notes on how to use various portions of the public SDK and is intended as a resource for developers creating software that targets the V5.

In comparison to a desktop operating system, the SDK takes on the role of kernel-to-userspace [syscalls](https://en.wikipedia.org/wiki/syscall). However, while traditional syscalls are implemented using the Supervisor Call `svc` instruction (this switches the CPU into the privileged Supervisor Mode and runs the kernel's SVC exception handler), the VEX SDK just uses normal function calls from user code into VEXos itself. This has the upside of cutting out the overhead of switching CPU modes, which is ideal for real-time applications like robot controllers. It also means user code runs in the same CPU mode as the kernel and is free to re-configure the processor, which is why normal operating systems don't do this.

## API Usage

Figuring out how to use the SDK to a desired effect can be difficult because of the lack of public documentation and source code. Luckily, there are a few resources publicly available that allow us to infer its API usage.

### VEXcode Header Files

The "VEX Robotics" (VEXcode) Visual Studio Code extension downloads header files for the SDK into its VS Code `globalStorage` folder. The location of this is platform-specific:

* Linux: `~/.config/Code/User/globalStorage`
* macOS: `~/Library/Application Support/Code/User/globalStorage`
* Windows: `~\AppData\Roaming\Code\User\globalStorage`

From there, the header files can be found at:

`./vexrobotics.vexcode/sdk/cpp/V5/V5_<DATE>/vexv5/include/v5_api*.h`

The utility and means of using certain APIs can be inferred by the names of the function and which arguments it takes. For example, the following function in `v5_api.h` seems to set the foreground color of the V5's display to the given color:

```c
// Display/graphics
void vexDisplayForegroundColor(uint32_t col);
```

### PROS Source Code

In cases where more context is needed, it's best to search for the SDK function's name in the [PROS Kernel source code](https://github.com/purduesigbots/pros) to see how they use it, or view the source of the PROS function with the desired functionality. The PROS developers were given access to internal VEX documentation about the SDK, so their usage of it can generally be considered canonical.

For example, the PROS function for moving a motor, `motor_move_voltage`, is effectively a wrapper for `vexDeviceMotorVoltageSet` with some extra mutex and negation logic tacked on. From this, one can infer that `vexDeviceMotorVoltageSet` outputs a certain voltage to the specified motor using the value given in **millivolts**.

```c
// Copyright (c) 2017-2024, Purdue University ACM SIGBots.
// This Source Code Form is subject to the terms of the Mozilla
// Public License, v. 2.0.

/**
 * Sets the output voltage for the motor from
 * -12000 to 12000 in millivolts.
 *
 * \note A negative port negates the voltage
 */
int32_t motor_move_voltage(int8_t port, int32_t voltage) {
    uint8_t abs_port = abs(port);
    claim_port_i(abs_port - 1, E_DEVICE_MOTOR);
    if (port < 0) voltage = -voltage;
    vexDeviceMotorVoltageSet(device->device_info, voltage);
    return_port(abs_port - 1, PROS_SUCCESS);
}
```

### VEX Resources

Sometimes VEX employees release public code samples that include the use of VEX SDK functions, or discuss the proper usage of the SDK on the VEX Forum.

For example:

* `vexGenericSerialReceive` was discussed in [this thread](https://www.vexforum.com/t/use-v5-smart-port-as-generic-serial-device-pros/57821/18?page=2).
* The usage of `vexDisplayClipRegionSetWithIndex` was shown in [this GitHub repo](https://github.com/jpearman/V5_CompetitionTest/blob/efb7214b983d30d5583e39b343161c26d7187766/include/comp_debug.h#L95).

A helpful search term to find these resources is: `"extern \"C\"" site:vexforum.com`.

## Jumptable

Since SDK functions are part of VEXos itself, every time the OS is re-compiled, their location in memory shifts around. If user programs tried to call the functions directly, they would all break after each VEXos update. Thus, VEXos includes an array of pointers to SDK functions (in other words, a table of jumps) at a constant location in memory which stays up to date between OS version changes.

```rs
// Example OS code. Even if things get moved around,
// JUMPTABLE will stay put in the ".jumptable" section.

extern "C" fn vexDoSomething() {
    // sdk function impl...
}

#[unsafe(link_section = ".jumptable")]
static JUMPTABLE: [*const (); 1234] = [
    // other sdk functions...
    vexDoSomething as *const (), // item number 123
    // other sdk functions...
];
```

User programs can then make SDK calls by reading the pointer to the desired function off the jumptable and calling it.

```rs
// example user code

const JUMPTABLE_ADDRESS: usize = 0x037f_c000;

fn callVexDoSomething() {
    const OFFSET: usize = 123 * size_of::<*const ()>();

    // Get pointer to jumptable item (constant between OS versions)
    let vexDoSomething_ptr = (JUMPTABLE_ADDRESS + OFFSET)
        as *const extern "C" fn();

    // Read the real location of function (changes)
    let vexDoSomething = unsafe { *vexDoSomething_ptr };

    vexDoSomething()
}
```

### Offsets

There's no official reference for which jumptable indexes match which SDK functions. However, this information can be extracted fairly easily from the library `libpros.a` which is distributed as part of the open-source, MPL-licensed [PROS kernel](https://github.com/purduesigbots/pros/releases/latest).

The `llvm-objdump` tool can be used to show the assembly for the desired SDK wrapper function:

<pre><code>$ llvm-objdump --disassemble-symbols=vexDeviceMotorVoltageSet --mcpu=cortex-a9 ./firmware/libpros.a

./firmware/libpros.a(v5_apijump.c.obj): file format elf32-littlearm

Disassembly of section .text.vexDeviceMotorVoltageSet:

00000000 &lt;vexDeviceMotorVoltageSet&gt;:
       0: e59f3004      ldr     <span style='color:var(--cyan,#0aa)'>r3</span>, <span style='color:var(--green,#0a0)'>[</span><span style='color:var(--cyan,#0aa)'>pc</span><span style='color:var(--green,#0a0)'>, </span><span style='color:var(--red,#a00)'>#0x4</span><span style='color:var(--green,#0a0)'>]</span>          @ 0xc &lt;vexDeviceMotorVoltageSet+0xc&gt;
       4: e593335c      ldr     <span style='color:var(--cyan,#0aa)'>r3</span>, <span style='color:var(--green,#0a0)'>[</span><span style='color:var(--cyan,#0aa)'>r3</span><span style='color:var(--green,#0a0)'>, </span><span style='color:var(--red,#a00)'>#0x35c</span><span style='color:var(--green,#0a0)'>]</span>
       8: e12fff13      bx      <span style='color:var(--cyan,#0aa)'>r3</span>
       c: 037fc000      cmneq   <span style='color:var(--cyan,#0aa)'>pc</span>, #<span style='color:var(--red,#a00)'>0</span>
</code></pre>

This can be interpreted as equivalent to the following pseudocode:

```rs
fn vexDeviceMotorVoltageSet(...args) {
/* 0: */    let jumptable_start = *&JUMPTABLE;
/* 4: */    let fn_pointer = *(jumptable_start + 0x35c);
/* 8: */    fn_pointer(...args)
}
/* c: */    static JUMPTABLE = 0x037fc000;
```

One can infer from this that the offset of `vexDeviceMotorVoltageSet`'s function pointer relative to the start of the jumptable array is `0x35c`.

When analyzing libraries related to the VEX SDK, you should keep in mind that certain first-party VEX products (e.g. `libv5rt.a`) have licenses which include clauses intended to prevent reverse engineering such as disassembly or decompilation. We believe these agreements do not apply to any of the actions described in these notes. However, we are not lawyers, this is not legal advice, and you should exercise due caution.
