# VEX V5 Environment Summary

## Hardware

- **System-on-chip**: Zynq 7000 ([Reference manual](https://docs.amd.com/viewer/book-attachment/mxcNFn1EFZjLI1eShoEn5w/~F6mCjSrb3_BDetD7782sA-mxcNFn1EFZjLI1eShoEn5w))
- **Processor**: Cortex A9 ([Reference manual](https://documentation-service.arm.com/static/5e8e2ab9fd977155116a7035?token=))
- **Architecture**: ARMv7-A ([Reference manual](https://developer.arm.com/documentation/ddi0406/latest))

(When developing system-level software for the V5 brain, the linked manuals are highly recommended resources.)

The VEX V5 is built around the Zynq 7000 system-on-chip, running user code on CPU1, the second of its two ARM Cortex-A9 processors. Code on CPU1 does not have an operating system in the traditional sense and is free to make priviledged changes to device state.

The Zynq 7000 has an onboard FPGA which is used for things like UART and controlling the device's LCD display. Its processors run the ARMv7-A architecture with the Security and Advanced SIMD extensions (but not the virtualization or large address space extensions). The Cortex-A9 has an MMU, which means it uses the Virtual Memory System Architecture.

## Software

User code execution begins at the address `0x0380_0020` in the *System* processor mode in *Secure* state. Since *System* is a PL1 (permission-level-one) mode, it has permissison to make operating-system-level changes to its environment: for instance, user code can set the system interrupt handler or access the system debug registers. This level of freedom is a feature intended to allow custom scheduling mechanisms and the sort. PROS uses these capabilities to run FreeRTOS directly on-device.

Programs are expected to [use the VEX SDK](../sdk/README.md) to interface with most peripherals. Some peripherals (e.g. smart devices) are controlled indirectly by CPU0, and the SDK handles sending the required control messages between the two CPUs. Periodically calling `vexTasksRun` allows this communication to take place smoothly - without it, sensors might not receive new values and buffered outputs might not flush.
