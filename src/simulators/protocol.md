# Vexide Simulator Protocol

Vexide Simulator Protocol allows simulator code executors and frontends to communicate. Software that implements it can work interchangeably.

## Specification

This specification details the Vexide Simulator Protocol, used by tools in the vexide ecosystem to achieve interoperability between simulators and their frontend. It is primarily intended to inform developers of libraries creating software compatible with tools utilizing the protocol.

### Requirements

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this section are to be interpreted as defined in [RFC 2119](https://tools.ietf.org/html/rfc2119.html).

### Overall Operation

Vexide Simulator Protocol (the "Protocol") is a newline-delimited JSON-based protocol intended to facilitate communication between robot simulators and frontends like GUIs. All communication MUST be in the [JSON Lines](https://jsonlines.org/) format.

Implementors of the Protocol SHALL recognize two unique roles in a given session: the Code Execution Backend (the "Simulator" or "Backend") and the Frontend. A Vexide Simulator Protocol session begins with the creation of an I/O stream between the Simulator and the Frontend. Implementors SHOULD support sessions over the [standard streams](https://en.wikipedia.org/wiki/Standard_streams) of the Simulator using its standard input and standard output, but MAY support other methods such as Unix domain sockets or Transmission Control Protocol (TCP) streams.

Throughout a given session, the Code Execution Backend and the Frontend send JSON messages over the underlying stream. Messages sent by the Backend are referred to as Events because they are used to notify the Frontend of changes in the simulator world. Messages sent by the Frontend are referred to as Commands because they are used to control the behavior of the Simulator.

#### Standard Stream Considerations

When using standard streams to communicate, data sent by the Backend over its standard error stream MUST be considered unstructured logs by the Frontend and SHOULD be made accessible to users. This requirement is made to aid the discoverability of Simulator error messages and improve the troubleshooting experience.

### Data Type Format

Events and commands sent using the Protocol contain various JSON data types analogous to [Rust type values](https://doc.rust-lang.org/reference/types.html). Implementors of the Protocol MUST decode and encode data types to a format compatible with the [Serde](https://serde.rs/json.html) library configured to use [externally tagged enums](https://serde.rs/enum-representations.html).

### Specific Commands and Events

The specific commands and events that implementors may use are currently defined in the [Commands](#commands) and [Events](#events) sections, respectively.

### Protocol Timeline

A session begins with the Frontend sending a `Handshake` command containing the maximum Protocol version it is compatible with (the current version is `1`), as well as an array of string IDs containing an unspecified list of extensions (i.e. deviations from the specification) that it is compatible with. If the Simulator is not compatible with the protocol version sent by the frontend, it SHOULD immediately close the stream, ending the session. Otherwise, it MUST send a `Handshake` event containing the protocol version and extensions that will be used henceforth. The protocol version MUST be less than or equal to the one sent by the Frontend, and the extensions array MUST only contain values that the Frontend indicated it was compatible with. Unknown fields in the handshake SHALL be ignored by all implementations.

The handshake is finished when both the Frontend and Backend have sent `Handshake` messages. Events and commands other than `Handshake` MAY NOT be sent until the handshake has finished.

After the handshake has finished, the Frontend MUST send `ConfigureDevice` commands for each peripheral that is already configured to make them available to the robot code. The Frontend SHOULD also send a `CompetitionMode` command containing the desired starting competition mode, as well as any other commands necessary to set the Simulator to the desired starting state.

If the Frontend does not send a `CompetitionMode` command, the Backend SHALL default to a competition mode with the following fields:

- `enabled`: `true`
- `connected`: `false`
- `mode`: `Driver`
- `is_competition`: `false`

Before continuing the session, the Backend MUST handle setup commands sent by the Frontend and at any point send a `Ready` event when it is capable of immediately beginning code execution. To continue the simulation, the Frontend MUST wait for the Backend's `Ready` event, and then send the `StartExecution` commands. The Backend SHOULD ignore `StartExecution` commands received before it has signaled it is ready.

After receiving the `StartExecution` command, the Backend SHALL take the steps necessary to begin executing robot code and SHOULD send events as necessary to notify the Frontend of changes to the state of the simulation.

If the robot code exits for any reason, the Simulator MUST send an `Exited` event, followed by ending the session by closing the underlying stream. If the code exited due to a fatal error, the Simulator SHOULD precede its `Exited` event with an `Error`-level `Log` event containing a concise explanation of what went wrong. If the Simulator ends the session by closing the underlying stream without first sending an `Exited` event, the Frontend SHOULD handle the situation as an internal error in the Simulator.

#### Code Signatures

Backends SHOULD send a `VCodeSig` event containing the robot program's Code Signature (i.e. "Cold Header") as soon as possible after the handshake has finished. Code signatures MUST be encoded in the Base64 format.

Here is the output of `hexdump -C` on a code signature commonly used by vexide programs:

```txt
00000000  58 56 58 35 02 00 00 00  00 00 00 00 00 00 00 00  |XVX5............|
00000010  00 00 00 00 00 00 00 00                           |........|
00000018
```

This code signature would be encoded by a compatible Simulator as the following Base64-encoded JSON string literal:

```json
"WFZYNQIAAAAAAAAAAAAAAAAAAAAAAAAA"
```

#### Logging

Backends MAY send human-readable `Log` events at any time after the handshake is completed. Frontends SHOULD make these logs available to users and MAY store them for later retrieval. Frontends MAY also implement filtering to reduce clutter caused by log levels that tend to be more verbose.

### Example Session

This example session is provided as a resource to help developers imagine one of the possibilities of a valid exchange using the Protocol. Multi-line JSON, as well as comments preceded by a double-slash (`//`), are not valid in a true implementation and are used to aid the reader in understanding.

```jsonc
// The Frontend begins the session by sending a handshake.
// Frontend:
{ "Handshake": { "version": 1, "extensions": [] } }
// The Backend agrees on using version 1 without extensions.
// Backend:
{ "Handshake": { "version": 1, "extensions": [] } }
//
// The Frontend begins configuring devices.
// Frontend:
{ "ConfigureDevice": {
  "port": 1,
  "device": { "Motor": {
    "physical_gearset": "Red",
    "moment_of_inertia": 1.0
  } }
} }
{ "ConfigureDevice": {
  "port": 2,
  "device": { "Motor": {
    "physical_gearset": "Green",
    "moment_of_inertia": 1.0
  } }
} }
// Meanwhile, the Backend has finished loading the robot program's code signature.
// Backend:
{ "VCodeSig": "WFZYNQIAAAAAAAAAAAAAAAAAAAAAAAAA" }
// The Frontend continues setup.
// Frontend:
{ "ConfigureDevice": {
  "port": 3,
  "device": { "Motor": {
    "physical_gearset": "Red",
    "moment_of_inertia": 2.0
  } }
} }
{ "CompetitionMode": {
  "enabled": true,
  "mode": "Driver",
  "connected": true,
  "is_competition": false
} }
// The Frontend has finished configuring devices and is now waiting for the
// Backend to be ready.
//
// The Backend is now ready to execute the robot code.
// Backend:
"Ready"
//
// The user has indicated the robot code should start.
// Frontend:
"StartExecution"
// The robot program has sent a line over the serial port.
// Backend:
{ "Serial": {
  "channel": 1,
  "data": "SGVsbG8gV29ybGQhCg=="
} }
// The robot program is moving a motor.
// Backend:
{ "DeviceUpdate": {
  "port": 1,
  "status": { "Motor": {
    "velocity": 1.0,
    "reversed": false,
    "power_draw": 1.0,
    "torque_output": 1.0,
    "flags": 0,
    "position": 2.5,
    "target_position": null,
    "voltage": 5.0,
    "gearset": "Red",
    "brake_mode": "Brake"
  } }
} }
// The user has changed the competition mode.
// Frontend:
{ "CompetitionMode": {
  "enabled": false,
  "mode": "Driver",
  "connected": true,
  "is_competition": false
} }
//
// The robot code has exited.
// Backend:
"Exited"
```

## Communication

The code executor and frontend talk over a stream in [newline-delimited JSON format](https://jsonlines.org/).

The backend sends [``Event``](#events)s which represent a change in simulator state.
These are used by the frontend to correctly display the state of the simulated program.

The frontend sends [``Commands``](#commands) to the code executor to control the robot code environment, simulating changes in robot hardware (like controller input and LCD touch events) or competition phase.

When using Standard I/O to communicate, data sent by the simulator engine over `stderr` must be considered unstructured logs and should be accessible to the user.

## Events

Events are sent from the simulator backend to the frontend to describe simulator state changes.

- `Handshake`: Specifies the version of the Protocol used, as well as extensions that the Backend is compatible with. Fields:
  - version: Nonzero integer. Compatible implementations MUST set this to the value `1`.
  - extensions: String array
- `ScreenDraw`: Draw something on the screen. `ScreenDraw` fields:
  - command: [DrawCommand](#drawcommand)
  - color: [``Color``](#color)
- `ScreenClear`: Fill the entire screen with one color. `ScreenClear` fields:
  - color: [``Color``](#color)
- `ScreenDoubleBufferMode`: Set the double-buffer mode of the screen. When double buffer mode is enabled draw calls should be stored in a intermediate buffer and flushed to the screen only once a `ScreenRender` event is sent. `ScreenDoubleBufferMode` fields:
  - enable: boolean
- `ScreenRender`: Flush the screens double buffer if screen double buffering is enabled.
- `VCodeSig`: The cold header ("Code Signature") of the robot program, encoded as a base-64 string.
- `Ready`: The backend is ready to start executing user code.
- `Exited`: The backend has stopped exiting user code and will be terminating.
- `Serial`: Data that has been flushed from the serial FIFO buffer. `Serial` fields:
  - channel: integer
  - data: Base64-encoded string
- `DeviceUpdate`: State regarding an ADI or Smart device has changed. `DeviceUpdate` fields:
  - status: [DeviceStatus](#devicestatus)
  - port: [Port](#port)
- `Battery`: New statistics about the current state of the robot battery are ready. Fields:
  - voltage: The output voltage of the battery. Float value in Volts.
  - current: The current draw of the battery. Float value in Amps.
  - capacity: The remaining energy capacity of the battery. Float value in Watt-Hours.
- `RobotPose`: The physically simulated robot has moved. `RobotPose` fields:
  - x: float
  - y: float
- `RobotState`: The state of the physically simulated robot has changed. Implementation is TBD.
- `Log`: Log a message. This is not to be used in place of serial. This is purely for messages from the backend itself. `Log` fields:
  - level: [LogLevel](#loglevel)
  - message: UTF8 encoded string.
- `VEXLinkConnect`: Tell the, currently nonexistent, VEXLink server to open a VEXLink connection. `VEXLinkConnect` fields:
  - port: [SmartPort](#smartport).
  - id: string (unhashed id)
  - mode: [LinkMode](#linkmode)
  - override: boolean
- `VEXLinkDisconnect`: Tell the VEXLink server that the VEXLink connection has been terminated. Fields:
  - port: [SmartPort](#smartport).

## Commands

Commands are sent from the frontend to the backend and signal for specific actions to be performed.

- `Handshake`: Specifies the version of the Protocol used, as well as extensions that the Frontend is compatible with. Fields:
  - version: Nonzero integer. Compatible implementations MUST set this to the value `1`.
  - extensions: String array
- `Touch`: Touch a point on the screen. Only one touch can be registered on the Brain display. `Touch` fields:
  - pos: [Point](#point)
  - event: [TouchEvent](#touchevent)
- `ControllerUpdate`: Updates the current state of the controller. A [ControllerUpdate](#controllerupdate) enum.
- `USD`: Mount or unmount a directory as the V5's SD Card. Robot programs will be able to read and write to this directory. Fields:
  - root: string (path to sd card's root), or null to unmount
- `VEXLinkOpened`: The VEXLink server has successfully opened a connection. Fields:
  - port: [SmartPort](#smartport).
  - mode: [LinkMode](#linkmode)
- `VEXLinkClosed`: The VEXLink server has closed a VEXLink connection. Fields:
  - port: [SmartPort](#smartport).
- `CompetitionMode`: Update the competition mode. Fields:
  - enabled: boolean
  - connected: boolean
  - mode: [CompMode](#compmode)
  - is_competition: boolean
- `ConfigureDevice`: Configure a port as a specified device. Fields:
  - port: [Port](#port)
  - device: [Device](#device)
- `AdiInput`: Set the input voltage for an ADI port. Implementation is TBD. Fields:
  - port: [AdiPort](#adiport)
  - voltage: float
- `StartExecution`: Start executing user code. This should be treated as a no-op by the backend until it sends a `Ready` [`Event`](#events).
- `SetBatteryCapacity`: Updates the remaining battery capacity to a new Watt-Hours value. If this command is not sent, the simulator must default to a value of 14 Watt-Hours, the maximum capacity of the VEX V5 battery. This value may be used by the simulator to help calculate the voltage of the battery. Fields:
  - capacity: float in Watt-Hours

## Data types

Several different enums and structs are used for communication to and from the backend.

All enum datatypes are encoded using externally tagged representation; see [Serde enum-representations](https://serde.rs/enum-representations.html).

### DrawCommand

An enum with these variants:

- `Fill`: Fill a shape on the screen.  Fields:
  - shape: [Shape](#shape)
- `Stroke`: Draw the outline of a shape on the screen. Fields:
  - shape: [Shape](#shape)
- `CopyBuffer`: Draw a pixel buffer to the screen with a given stride and start and ending coordinates. `CopyBuffer` fields:
  - top_left: [Point](#point)
  - bottom_right: [Point](#point)
  - stride: nonzero integer
  - buffer: image buffer of 32-bit RGB pixels as a base64-encoded string

### Shape

An enum that describes a shape to be drawn on the screen.
Shape has these variants:

- `Rectangle`: Rectangles are drawn starting at the top left coordinate and extending to the bottom right coordinate. `Rectangle` fields:
  - top_left: [Point](#point)
  - bottom_right: [Point](#point)
- `Circle`: `Circle`s are drawn with a coordinate at the center of the circle and a radius. This variant has these fields:
  - center: [Point](#point)
  - radius: integer
- `Pixel`: Draw a single pixel at a coordinate. `Pixel` fields:
  - pos: [Point](#point)

### Point

A struct that stores a pixel coordinate.
The origin is at the top left of the screen.
``Point`` fields:

- x: integer
- y: integer

### DeviceStatus

An enum representing the state of a device.
Every time the state of a device on any port changes, this enum will be sent.
This type should be considered "non-exhaustive" and variants may be added without causing a breaking change.
``DeviceStatus`` variants:

- ``Motor``: The state of the motor has changed.
  ``Motor`` fields:
  - velocity: float in radians per second.
  - reversed: boolean
  - power_draw: float in Watts
  - torque_output: float in Nm
  - flags: A 32-bit integer bitfield containing the motor flags that will be provided to the robot code. VEX V5 uses this to signal motor faults. Implementors should set this to `0`.
  - position: float in radians
  - target_position: optional float in radians
  - voltage: float in Volts.
  - gearset: [``MotorGearSet``](#motorgearset)
  - brake_mode: [``MotorBrakeMode``](#motorbrakemode)

### MotorGearSet

Represents the gearset of a smart motor device.
Variants:

- ``Red``: 36-1 ratio
- ``Green``: 18-1 ratio
- ``Blue``: 6-1 ratio

### MotorBrakeMode

Represents the brake mode of a smart motor device.
Variants:

- ``Coast``
- ``Brake``
- ``Hold``

### MotorFlags

TBD

### LinkMode

An enum representing the type of VEXLink connection.
`LinkMode` variants:

- `Manager`
- `Worker`

### TouchEvent

An enum representing how a touch on the Brain display has changed.
`TouchEvent` variants:

- `Released`: The screen is no longer being pressed.
- `Pressed`: A new touch has been registered on the screen or the screen has been held but not long enough to start sending `Held` events.
- `Held`: Sent in place of `Pressed` events once a timeout has been reached. I cannot find the exact timeout for this, but it can easily be tested with a simple program.

### Port

An enum containing a Smart port or ADI port.
`Port` variants:

- `Smart`: [SmartPort](#smartport)
- `Adi`: [AdiPort](#adiport)

### SmartPort

An integer with the constraints 0 ≤ integer ≤ 20.

### AdiPort

An integer with the constraints 0 ≤ integer ≤ 7.

### CompMode

An enum representing the current phase of competition.
Variants:

- `Auto`
- `Driver`

### Color

A struct that represents an rgb8 color.
`Color` fields:

- r: 8 bit integer
- g: 8 bit integer
- b: 8 bit integer

### LogLevel

An enum representing the importance of a log message.
Variants:

- `Trace`: jumptable calls and other verbose messages
- `Info`: non-critical informational messages
- `Warn`: possible issues or errors that do not directly affect the simulation
- `Error`: issues and errors that degrade the simulation

### ControllerUpdate

An enum representing a method of accessing a controller's status. If the server recieves a ControllerUpdate command while in a disabled or autonomous mode, it must store the state recieved even though the robot code cannot access it yet.
Variants:

- `Raw`: [ControllerState](#controllerstate). If this variant is recieved, the server must stop updating the controller's status using a previously recieved UUID and only provide the robot code with the most recently recieved raw state.
- `UUID`: string, the UUID of a hardware gamepad such that it can be accessed via SDL2. If this variant is recieved, the server must discard any previously recieved `Raw` data and continue updating its internal representation of the controller using the specified UUID even if no more `ControllerUpdate` commands are sent.

### ControllerState

The current state of a controller's inputs.
Fields:

- `axis1`: integer with range of `[-127, 127]`
- `axis2`: integer with range of `[-127, 127]`
- `axis3`: integer with range of `[-127, 127]`
- `axis4`: integer with range of `[-127, 127]`
- `button_l1`: boolean
- `button_l2`: boolean
- `button_r1`: boolean
- `button_r2`: boolean
- `button_up`: boolean
- `button_down`: boolean
- `button_left`: boolean
- `button_right`: boolean
- `button_x`: boolean
- `button_b`: boolean
- `button_y`: boolean
- `button_a`: boolean
- `button_sel`: boolean
- `battery_level`: integer with range of `[0, battery_capacity]`
- `button_all`: boolean
- `flags`: A 32-bit integer bitfield containing the controller flags that will be provided to the robot code. Implementors should set this to `0`.
- `battery_capacity`: positive integer

### Device

An enum of the configurations for a peripheral device.
This type should be considered "non-exhaustive" and variants may be added without causing a breaking change.
Variants:

- `Motor`: The configuration for a motor peripheral. Fields:
  - `physical_gearset`: [``MotorGearSet``](#motorgearset)
  - `moment_of_inertia`: float in kilogram meters squared. Controls the acceleration of the motor when it applies a set amount of torque.
