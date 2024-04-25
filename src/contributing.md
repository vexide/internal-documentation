# Contributing

This chapter aims to give tips to first time vexide contributors.
If you want to contribute to vexide, make sure you [join the discord server](https://discord.com/invite/8WR4X4AGVg)!
For info on contributing guidelines, please read [CONTRIBUTING.md](https://github.com/vexide/vexide/blob/main/CONTRIBUTING.md) in the vexide repository.

## Note

All development on pros-rs has stopped.
We are now solely developing and maintaining vexide.

## Project structure

Because vexide is split into multiple crates and multiple repositories,
finding the part of vexide you want to work on can be a little overwhelming.
To hopefully make it easier to find what you are looking for,
here is a simple graph of the project structure:

- [vexide](https://github.com/vexide/vexide)
  - packages
    - vexide
    - vexide-async
    - vexide-core
    - vexide-devices
    - vexide-macro
    - vexide-math
    - vexide-panic
    - vexide-startup
- [pros-rs](https://github.com/vexide/pros-rs)
  - packages
    - [pros](https://crates.io/crates/pros)
    - [pros-sys](https://crates.io/crates/pros-sys)
    - [pros-async](https://crates.io/crates/pros-async)
    - [pros-core](https://crates.io/crates/pros-core)
    - [pros-devices](https://crates.io/crates/pros-devices)
    - [pros-math](https://crates.io/crates/pros-math)
    - [pros-panic](https://crates.io/crates/pros-panic)
    - [pros-sync](https://crates.io/crates/pros-sync)
- [cargo-pros](https://github.com/vexide/cargo-pros) (vexide build tooling with the `cargo pros` command)
- [pros-simulator](https://github.com/vexide/pros-simulator)
  - packages
    - [pros-simulator](https://crates.io/crates/pros-simulator) (WASM simulator backend)
    - [pros-simulator-interface](https://crates.io/crates/pros-simulator-interface)
    - [pros-simulator-server](https://crates.io/crates/pros-simulator-server) (Standalone wrapper over the wasm backend)
- [vex-v5-sim](https://github.com/vexide/vex-v5-sim) (WIP QEMU simulator backend)
- [internal-documentation](https://github.com/vexide/internal-documentation) (This website)
- [website](https://github.com/vexide/website) (Our [main page](https://pros.rs))
