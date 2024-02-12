# Contributing

This chapter aims to give tips to first time pros-rs contributors.
For info on contributing guidelines, please read [CONTRIBUTING.md](https://github.com/pros-rs/pros-rs/blob/main/CONTRIBUTING.md) in the pros-rs repository.

## Project structure

Because pros-rs is split into multiple crates and multiple repositories,
finding the part of pros-rs you want to work on can be a little overwhelming.
To hopefully make it easier to find what you are looking for,
here is a simple graph of the project structure:

- [pros-rs](https://github.com/pros-rs/pros-rs)
  - packages
    - [pros](https://crates.io/crates/pros)
    - [pros-sys](https://crates.io/crates/pros-sys)
- [cargo-pros](https://github.com/pros-rs/cargo-pros) (Pros-rs build tooling with the `cargo pros` command)
- [pros-simulator](https://github.com/pros-rs/pros-simulator)
  - packages
    - [pros-simulator](https://crates.io/crates/pros-simulator) (WASM simulator backend)
    - [pros-simuator-interface](https://crates.io/crates/pros-simulator-interface)
    - [pros-simulator-server](https://crates.io/crates/pros-simulator-server) (Standalone wrapper over the wasm backend)
- [vex-v5-sim](https://github.com/pros-rs/vex-v5-sim) (Currently unimplemented QEMU simulator backend)
- [internal-documentation](https://github.com/pros-rs/internal-documentation) (This website)
- [website](https://github.com/pros-rs/website) (Our [main page](https://pros.rs))

The structure of pros-rs is very subject to change.
For instance PR [#86](https://github.com/pros-rs/pros-rs/pull/86) on the pros-rs repo will completely restructure the project.