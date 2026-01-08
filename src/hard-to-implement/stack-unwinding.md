# Stack Unwinding

Note: Since this page was written, the Rust std library was ported to VEXos. It is possible to implement stack unwinding by configuring libstd builds to depend on LLVM libunwind and use it for unwinding panics by changing around some `cfg`'s. However, at the time of writing, this hasn't been done by anybody. It's also unclear what the implications of this change would be on vexide's memory soundness and executable size.

When a Rust program encounters an unrecoverable error, it `panic!()`s. The behavior of a panic is intentionally not defined in Rust, and may vary based on the platform and strategy.

In general, Rust defines two "panic strategies":

- `abort`: After printing a stacktrace (defined by a `#[panic_handler]`), the program will exit immediately. No memory cleanup is performed. This form of panic is *truly* unrecoverable.
- `unwind`: Rather than exiting immediately, the program's stack memory will be *unwound*, allowing for a more graceful exit. Every active struct instance's `Drop` implementation will be called, and panics created outside of the main thread can be [caught and handled](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html).

## `no_std` and Panics: The current approach.

Panic behavior is platform-dependent. On a more "traditional" platform target, we have the luxury of an operating system with I/O utilities and a well-defined allocator. On an embedded target such as `armv7a-VEXos-eabi` (the platform target defined by `vexide`), Rust takes no assumptions and leaves the panic implementation up to us.

In order to build a barebones Rust program on bare metal, we must define a panic handler:

```rs
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    loop {}
}
```

In this (extremely barebones) example, if `panic!()` were to be called, the panic handler would simply spinlock the CPU indefinitely. If this were an enviornment where `libc` was available (PROS has access to newlib libc) we could call `libc::exit(1)`.

vexide currently defines a basic panic handler under the assumption of the `abort` strategy:

- It prints a panic message over stdout through serial.
- It optionally displays a brief error message on the brain screen.
- Finally, it exits the user program using PROS' wrapper over over the `vexSystemExitRequest()` SDK call.

The panic implementation does not perform stack unwinding or any cleanup, as `armv7a-VEXos-eabi`'s panic_strategy is currently `abort`.

## Unwinding on ARM

As it turns out, it might be possible to support `"panic_strategy": "unwind"` on our bare metal ARM target. PROS links against `libgcc`, which defines primitives for dealing with exception handling, including the parts needed to implement an unwinding panic handler in Rust. Unfortunately, stack unwinding has a very complicated and advanced implementation. The best we have to go off of is the [`libpanic` implementation over `libgcc`](https://github.com/rust-lang/rust/blob/c57393e4f8b88444fbf0985a81a2d662862f2733/library/std/src/sys/personality/gcc.rs) used internally by rustc. This piece of code is a pretty obscure part of Rust's runtime. It's not well documented and is supposedly reverse-engineered from other language implementations.

Essentially, in order to implement an unwinding panic, we must first override the `eh_personality` language item which is called by Rust internally when panicking using `unwind`. The `#[eh_personality]` routine is then responsible for hooking into libgcc's ARM unwinding functions for actually unwinding the stack. These functions are actually part of the [Itanium C++ Exception Handling ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html).

In a [previous attempt](https://github.com/vexide/pros-rs/compare/main...Tropix126:pros-rs:feat/panic-unwind) to reimplement unwinding panics, that item looked something like this:

```rs
#![feature(lang_items)]

#[lang = "eh_personality"]
#[no_mangle]
unsafe extern "C" fn rust_eh_personality(
    state: _Unwind_State,
    exception_object: *mut _Unwind_Exception,
    context: *mut _Unwind_Context,
) -> _Unwind_Reason_Code {
  ...
}
```

PROS Additionally has its own [unwinding exception handler](https://github.com/purduesigbots/pros/blob/master/src/system/unwind.c).

## Other Alternatives

The [`unwinding`](https://crates.io/crates/unwinding) crate provides pure-rust implementations of exception handlers that we could use in place of a libgcc implementation. Unfortunately, ARM support isn't available in this crate yet.
