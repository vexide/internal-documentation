# Critical Section

vexide has a critical section implementation that is used to ensure that synchonization types are safe from interrupts.

We use the [``critical_section``](https://crates.io/crates/critical-section) so that libraries that require a critical section implementation can easily access it.

When building for WASM, the critical section implementation is replaced with no-ops because the critical section is implemented using ARM assembly.

## Implementation

When entering the critical section we want to disable all IRQ interrupts.
What this means is that nothing can interrupt the flow of our code and mess with memory that needed to be untouched.

The first step that vexide takes when entering the critical section is checking if interrupts are already disabled.
```rust
    let mut cpsr: u32;
    asm!("mrs {0}, cpsr", out(reg) cpsr);
    let masked = (cpsr & 0b10000000) == 0b10000000;
```
This block of code moves the value of the ``cspr`` register into the cspr variable and then checks if the 7th bit is a 1.
If it is 1, IRQ interrupts are masked.
It is important to know if interrupts are masked because nested critical sections shouldn't re-enable interrupts if an inner section is exited.