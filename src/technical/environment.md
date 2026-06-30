# VEX V5 Environment Summary

## Hardware

- **System-on-chip**: Zynq 7000 ([Reference manual](https://docs.amd.com/r/en-US/ug585-zynq-7000-SoC-TRM/), [PDF](Zynq7000_TRM_v1.14.pdf))
- **Processor**: Cortex A9 ([Reference manual](https://documentation-service.arm.com/static/5e8e2ab9fd977155116a7035?token=))
- **Architecture**: ARMv7-A ([Reference manual](https://developer.arm.com/documentation/ddi0406/latest))

(When developing system-level software for the V5 brain, the linked manuals are highly recommended resources.)

The VEX V5 is built around the Zynq 7000 system-on-chip, running user code on CPU1, the second of its two ARM Cortex-A9 processors. Code on CPU1 does not have an operating system in the traditional sense and is free to make privileged changes to device state.

The Zynq 7000 has an onboard FPGA which is used for things like UART and controlling the device's LCD display. Its processors run the ARMv7-A architecture with the Security and Advanced SIMD extensions (but not the virtualization or large address space extensions). The Cortex-A9 has an MMU, which means it uses the Virtual Memory System Architecture.

## Software

User code execution begins at the address `0x0380_0020` in the *System* processor mode in *Secure* state. Since *System* is a PL1 (permission-level-one) mode, it has permission to make operating-system-level changes to its environment: for instance, user code can set the system interrupt handler or access the system debug registers. This level of freedom is a feature intended to allow custom scheduling mechanisms and the like. PROS uses these capabilities to run FreeRTOS directly on-device.

Programs are expected to [use the VEX SDK](../sdk/) to interface with most peripherals. Some peripherals (e.g. smart devices) are controlled indirectly by CPU0, and the SDK handles sending the required control messages between the two CPUs. Periodically calling `vexTasksRun` allows this communication to take place smoothly - without it, sensors might not receive new values and buffered outputs might not flush.

### System Configuration

This section contains the initial values of some common system registers.

```txt
SCTLR: 0b00001000110001010001100001111101
```

### Stack Sizes

([Details](stack_sizes.txt))

VEXos initializes each CPU mode's stack pointer to a smallish stack outside of normal user memory. If you need more space, you might need to allocate your own stack in user memory, then switch to it on startup. Here are the sizes and addresses of the default stacks:

| Stack | Size | Start | End |
| ----- | ---- | ----- | --- |
| User/System | `0x2000` (8 KiB) | `0x0371C090` | `0x0371E090` |
| IRQ | `0x2000` (8 KiB) | `0x0371E090` | `0x03720090` |
| Supervisor | `0x0800` (2 KiB) | `0x03720090` | `0x03720890` |
| Abort | `0x0400` (1 KiB) | `0x03720890` | `0x03720C90` |
| FIQ | `0x0400` (1 KiB) | `0x03720C90` | `0x03721090` |
| Undefined | `0x0400` (1 KiB) | `0x03721090` | `0x03721490` |
