# How Hydrozoa runs a program

## Step 1: Language build tools

Hydrozoa relies on a language's own build tools to create a WebAssembly (WASM) binary that it can load when the user starts their program. We chose WASM because of its wide language support - everything from Rust to JavaScript can be compiled to it. The VEX brain can't run WASM bytecode out of the box, so we include a separate interpreter which does.

In the case of Java, the language was designed to target the Java Virtual Machine (JVM) rather than WASM. This means programs need to use an alternative compiler. When using the Hydrozoa Gradle plugin, the [TeaVM](https://teavm.org) compiler's setup & invocation is handled automatically while running the `build` or `upload` tasks.

## Step 2: Uploading

Hydrozoa programs are bigger than C++ or Rust programs because they include a full interpreter and its bindings to the VEX SDK, not to mention the actual user code itself. Thus, the upload process is split into two files: the runtime and the user program.

The runtime contains everything that usually doesn't need to be reuploaded when a program changes, which includes the interpreter, VEX startup code, bindings to the VEX SDK, and the TeaVM support library. It currently exists on the brain as a single `libhydrozoa.bin` file, although this is subject to change.

In contrast, the user program contains WASM-formatted data that the runtime is expected to load. While there is only one Hydrozoa runtime per brain, there can be multiple user programs in different program slots. While uploading a program with the `hydrozoa` CLI, these upload progress for the runtime is marked with `lib` and the progress of the user program is marked with `bin`.

There tend to be very limited size constraints when uploading to the brain. The `wasm-opt` tool from Binaryen can help programmers meet these requirements, shrinking the program when running it like so:

```shell
wasm-opt -Oz -o optimized.wasm robot.wasm
```

This is not run automatically during a Gradle upload because then Binaryen would need to be installed manually by the user.

## Step 3: Startup

Hydrozoa uses a version of vexide-startup which has a modified linker script to support VEXos file linking. When the user presses the Run button,

1. VEXos loads the runtime to `0x03800000` and the user program to `0x07800000`.
2. It then begins execution at `0x03800020`, which is handled by vexide-startup.
3. After vexide-startup passes execution to the runtime, Hydrozoa reads the program from memory, initializes the interpreter (which is currently wasm3), and begins execution.

### User program format

VEXos does not include the length of a file when linking it, so the CLI prepends the user program with an integer equal to the program's length. The runtime knows about this format and uses it to retrieve the linked WASM file as a Rust `&[u8]` slice.

```txt
        0x07800000
            │
            ▼
┌──────────┐┌───────┐┌───────────────────┐
│End of    ││Program││WASM user program  │
│Runtime   ││Header ││(length: value of  │
│          ││       ││program header).   │
│          ││(u32)  ││                   │
│          ││       ││                   │
│          ││       ││                   │
│          ││       ││                   │
│          ││       ││                   │
└──────────┘└───────┘└───────────────────┘
            ▲
            │
```

## Step 4: Execution

A robot program is only useful if it can do something with its hardware, so Hydrozoa's runtime provides functions which WASM programs can import and use in order to access peripherals.

### The "vex" module

Functions imported from the `"vex"` module are used to access VEX peripherals and control the VEX robot. They are generally direct recreations of VEX SDK functions from the `vex-sdk` Rust crate and are likely similar to those defined in `libv5rts.a`.

For example, the following Rust program could theoretically be compiled to WASM, and when run under Hydrozoa, would draw a square to the VEX's display and then immediately exit:

```rs
#[link(wasm_import_module = "vex")]
unsafe extern "C" {
    fn vexDisplayForegroundColor(col: u32);
    fn vexDisplayRectFill(x1: i32, y1: i32, x2: i32, y2: i32);
}

#[unsafe(no_mangle)]
extern "C" fn start() {
    unsafe {
        vexDisplayForegroundColor(0xFFFFFF);
        vexDisplayRectFill(20, 20, 120, 120);
    }
}
```

While most signatures exported from the `"vex"` module are one-to-one with their `vex-sdk` counterpart, ones that deal in pointers have been modified for the unique runtime environment. For example, the functions that take a `V5_DeviceT` now take a `u32`. If the WASM program attempts to cast this number to a pointer and dereference it, the operation will fail, because WASM is executed in a memory sandbox.

WASM programs must periodically call the `vexTasksRun` function from the `"vex"` module in order to update sensors, perform basic serial I/O, and ensure the correct operation of the runtime.

```rs
#[link(wasm_import_module = "vex")]
unsafe extern "C" {
    fn vexDeviceGetByIndex(index: u32) -> u32;
    fn vexDeviceMotorVoltageSet(device: u32, voltage: i32);
    fn vexTasksRun();
}

#[unsafe(no_mangle)]
extern "C" fn start() {
    unsafe {
        // Get the motor on smart port 1
        let device_handle: u32 = vexDeviceGetByIndex(0);
        // Set it to 12 volts (full power)
        vexDeviceMotorVoltageSet(device_handle, 12 * 1000);
        // Loop so that the program doesn't exit
        loop {
            vexTasksRun();
        }
    }
}
```
