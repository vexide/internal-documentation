# Vexide Simulator Interface

Vexide Simulator Interface is a protocol that allows simulator code executors and frontends to communicate. Software that implements it can work interchangeably, similarly to the Language Server Protocol.

## Communication

The code executor and frontend talk over a stream in [newline-delimited JSON format](https://jsonlines.org/).

The code executor sends events that happen inside the simulator that the frontend might want to know about. Use this to monitor robot code progress, simulated LCD updates, log messages, and more.

The frontend sends messages to the code executor to control the robot code environment, simulating changes in robot hardware (like controller input and LCD touch events) or competition phase.

## Example

```json
// sim <-> frontend
// Code executor tells frontend it is booting
/* -> */ "Loading"
// Code executor tells frontend it has finished booting
// and is ready to know about the connected peripherals
/* -> */ "ResourcesRequired"
// Frontend tells code executor there is only a motor connected to port 1.
/* <- */ {"PortsUpdate":{"1":"Motor"}}
// Frontend tells code executor to start running the robot code
/* <- */ "Start"
// Code executor says it is running the robot code
/* -> */ "SchedulerStarted"
// Code executor tells frontend that the robot code wants to run Motor 1 at 1V.
/* -> */ {"MotorUpdated":{"port":1, "volts":1, "encoder_units": ..., ... }}
// Code executor tells frontend that the robot code has exited without error.
/* -> */ "AllTasksFinished"
```

## Notes

- API documentation available at <https://docs.rs/pros-simulator-interface>.
- `pros-simulator-server` only uses Stdio to communicate, but other streams are possible.
- Data over Stderr should be considered unstructured logs and may be shown to the user. As such, if the code executor crashes, its error message will be visible.
