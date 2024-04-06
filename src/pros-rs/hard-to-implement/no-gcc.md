# GCC independance

The pros-rs build process (the `armv7a-vexos-eabi` target and `cargo-pros`) depends on the Arm GNU toolchain (`arm-none-eabi-gcc`)
for linking and transforming output ELF executables
produced by Rust into binaries that can be run on a brain.
This dependency is less than ideal because the Arm GNU toolchain is **BIG**.
Uncompressed, it's about 1.7GiB which is pretty abysmal.
Ideally we would be using only LLVM `binutils` and `cargo-binutils`
as most users will already have these installed if they are using Rust.

Attempts have been made at using only llvm for linking, stripping, and `objcopy`ing
(see PR [#84](https://github.com/pros-rs/pros-rs/pull/84)).
Ultimately all attempts have failed because of a `libgcc` dependent unwinding implementation in PROS
and strange, likely linker related, memory access violation issues in FreeRTOS task switching code.
It could still technically be done if we manually patched `libpros`'
unwinding implementation to be indapendent of `libgcc`
and figured out what what caused the memory access violations,
however this would be difficult and would require a huge amount of testing.

If you are up to the the task of getting a pros-rs project running on a brain without `arm-none-eabi-gcc` or libgcc,
please take a shot at it. Getting this working would be amazing.

## TLDR

In order to not depend on GCC or `libgcc` we need to:
* patch `libpros` with a custom unwinding solution
* get programs linked with LLVM tools to not segfault
* strip everything related to `libgcc` from `libpros`
* update the `V5` linker scripts to work with LLVM

PR [#84](https://github.com/pros-rs/pros-rs/pull/84) does a couple of these things,
so check it out if you want to try getting this working.