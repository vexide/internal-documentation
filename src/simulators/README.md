# VEX Simulators

VEX V5 simulators are useful for debugging user programs without access to a real robot. This chapter describes the vexide/pros-rs ecosystem of simulator-related applications and how to write compatible software.

## Components of a Simulator

The following components are the most essential parts of a VEX simulator. Applications might implement one or more of these, or fill a supporting role not listed.

## Code Execution Backend (the "Simulator")

The code executor is probably the most important part of the simulator. Its job is to interpret user programs along with inputs like joystick controls and peripherals state in order to decide what the robot would do in real life (serial output, peripherals output, warnings, panics, etc). It might be implemented using a virtual machine, using a WebAssembly engine, or through dynamic library loading.

There are several code executors for VEX simulators, in varying stages of completion. Here are a few of the ones I know about:

- [LemLib/v5-sim-engine](https://github.com/LemLib/v5-sim-engine)

  This code executor, physics engine, and frontend combo uses a modified version of the PROS kernel to compile and run the robot code as a native executable. It is not currently being maintained.

- [vexide/pros-simulator: Run PROS robot code without the need for real VEX V5 hardware.](https://github.com/vexide/pros-simulator)

    This code executor compiles vexide robot code to WebAssembly and runs it in a sandbox. It, as well as vexide, are not currently being maintained.

- [vexide/vex-v5-sim: A simulator for the VEX V5 Brain. AKA The Vimulator.](https://github.com/vexide/vex-v5-sim)

    A unique, QEMU-based code executor, this project aims to accurately emulate a VEX V5 brain.

- There is a proof-of-concept WebAssembly code executor for Vexide.

## Physics Backend

This is a piece of software that decides robot and peripheral state (including velocity, temperature, etc.). It tells the code execution backend about the world state so that the simulated robot code can react to it.

## Frontend

The simulator frontend is usually the only part the user sees. It combines information from the physics backend and output from the code executor to show and overview of the simulation's progress. It also sends user input (e.g. joystick/touch) to the code executor to act accordingly.

- [vexide/pros-simulator-gui](https://github.com/vexide/pros-simulator-gui)

  A cross-platform Tauri application, it uses pros-simulator's frontend interface to start and manage a compatible simulator.
