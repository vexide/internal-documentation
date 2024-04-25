# Critical Section

vexide has a critical section implementation that is used to ensure that synchonization types are safe from interrupts.

We use the [``critical_section``](https://crates.io/crates/critical-section) so that libraries that require a critical section implementation can easily access it.

When building for WASM, the critical section implementation is replaced with no-ops because the critical section is implemented using ARM assembly.

## Implementation

When entering the critical section we want to disable all IRQ interrupts. What this means is that nothing can interrupt the flow of our code and run stuff in an order that is unsafe.

WIP