# Contributing

This chapter aims to give tips to first time vexide contributors.

If you want to contribute to vexide, make sure you [join the discord server](https://discord.com/invite/8WR4X4AGVg)!

For info on contributing guidelines, please read [CONTRIBUTING.md](https://github.com/vexide/vexide/blob/main/CONTRIBUTING.md) in the vexide repository.

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
        - vexide-panic
        - vexide-startup
- [](https://crates.io/crates/pros-sync)[cargo-v5](https://github.com/vexide/cargo-pros) (vexide build tooling with the `cargo v5` command)
- [vex-v5-qemu](https://github.com/vexide/vex-v5-qemu) (WIP QEMU simulator backend)
- [internal-documentation](https://github.com/vexide/internal-documentation) (This website)
- [website](https://github.com/vexide/website) (Our [main page](https://vexide.dev))
