# Task Scheduler

VEXos has a builtin cooperative task scheduler which is used to maintain internal state, coordinate communication with devices, and periodically run custom user-specified actions. When a user program begins, many system tasks are already running. The scheduler supports two types of tasks: simple tasks and full tasks.

## Task Types

**Simple tasks** are based around a callback and a next-run-timestamp. The scheduler tracks when each simple task is due to next run and invokes the callback each time its loop interval has elapsed. Most builtin system tasks are Simple tasks.

**Full tasks** behave more like OS threads: each has its own stack and supports sleeping. These are used to implement VEXcode's threads and tasks API. They are most useful when it'd be too difficult to break a long-running task into a state machine.

Full tasks were initially created for VEXcode's `vex::task` API (a [replacement](https://www.vexforum.com/t/how-to-use-vex-threads/100901/26?u=lewis) for the ROBOTC scheduler) but were later also used as the implementation for `vex::thread`, their version of C++'s `std::thread` API. Both classes do effectively the same thing under the hood.

## Stability

The task scheduler is considered a perma-unstable API by VEX, and, as such, the functions for controlling it are stripped out of the VEX [Partner SDK](/sdk/#partner-sdk). From the Partner SDK, the task scheduler is treated as an implementation detail of the `vexBackgroundProcessing` function, which is internally made to call `vexTasksRun` (and is a no-op on the public SDK).

Although unstable, the scheduler is used internally in VEXcode to implement its high-level multithreading API. It's unlikely the scheduler will see any breaking changes in the future because they would cause all existing compiled VEXcode programs to stop working.

## Ticking the Scheduler

By default, user code has full control of when/what code is run: to tick the task scheduler, it must periodically call `vexTasksRun`, which will run any simple task callbacks whose loop interval has elapsed and tick any full tasks that have finished sleeping. Since most system data is updated via tasks, programs should aim to tick the scheduler at least once every ~2ms to ensure they are operating on the latest data.

In programs that use high level frameworks (vexide, PROS, VEXcode), ticking the scheduler is done indirectly by sleeping the current task, which has led to the [widely recognized rule](https://pros.cs.purdue.edu/v5/tutorials/topical/multitasking.html) that all loops should contain a sleep.

Failing to tick the scheduler can lead to adverse effects such as printed text not appearing and device data not updating. It will not, however, prevent the robot from disabling itself after a match ends (since the competition lifecycle is entirely handled by CPU0).

## System Tasks

The following features are known to be implemented using the task scheduler:

- Receiving new device data: CPU0 is responsible for smart device communication, and a CPU1 task periodically copies the latest readings from CPU0 via shared memory.
  - This includes all ADI port communication because the V5 just uses a builtin ADI Expander smart device.
- Outbound USB serial data: writing to an outbound serial channel writes to a buffer of pending bytes and a task periodically flushes it to memory shared with CPU0.
  - Thus, printing a line of text to a serial port won't actually take effect until you run the task scheduler.
  - Notably, inbound USB serial data is [handled differently](https://www.vexforum.com/t/get-input-without-running-the-scheduler/146925/2?u=lewis): it's read directly from a FIFO ringbuffer in shared memory that's continuously filled by CPU0. It is reportedly safe to use from an interrupt handler.
- Ticking user touchscreen callbacks (`vexTouchUserCallbackSet`), but they can also be ticked manually by calling `vexTouchDataGet`.
