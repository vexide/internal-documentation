# Vexide Simulator Protocol

Vexide Simulator Protocol allows simulator code executors and frontends to communicate. Software that implements it can work interchangeably.

## Communication

The code executor and frontend talk over a stream in [newline-delimited JSON format](https://jsonlines.org/).

The backend sends [``Event``](#events)s which represent a change in simulator state.
These are used by the frontend to correctly display the state of the simulated program.

The frontend sends [``Commands``](#commands) to the code executor to control the robot code environment, simulating changes in robot hardware (like controller input and LCD touch events) or competition phase.

## Events

Events are sent from the simulator backend to the frontend to describe simulator state changes.

- `ScreenDraw`: Draw something on the screen. `ScreenDraw` fields:
  - command: [DrawCommand](#drawcommand)
  - color: [``Color``](#color)
- ScreenClear: Fill the entire screen with one color. `ScreenClear` fields:
  - color: [``Color``](#color)
- `ScreenDoubleBufferMode`: Set the double-buffer mode of the screen. When double buffer mode is enabled draw calls should be stored in a intermediate buffer and flushed to the screen only once a `ScreenRender` event is sent. `ScreenDoubleBufferMode` fields:
  - enable: boolean
- `ScreenRender`: Flush the screens double buffer if screen double buffering is enabled.
- `VCodeSig`: Sends info about the program header. Implementation details surrounding how this should be stored are TBD.
- `Ready`: The backend is ready to start executing user code.
- `Exited`: The backend has stopped exiting user code and will be terminating.
- `Serial`: Data that has been flushed from the serial FIFO buffer. `Serial` fields:
  - channel: integer
  - data: byte array
- `DeviceUpdate`: State regarding an ADI or Smart device has changed. `DeviceUpdate` fields:
  - status: [DeviceStatus](#devicestatus)
  - port: [Port](#port)
- Battery (implementation details not completely decided)
- `RobotPose`: The physically simulated robot has moved. `RobotPose` fields:
  - x: float
  - y: float
- `RobotState`: The state of the physically simulated robot has changed. Implementation is TBD.
- `Log`: Log a message. This is not to be used in place of serial. This is purely for messages from the backend itself. `Log` fields:
  - level: [LogLevel](#logLevel)
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

- `Touch`: Touch a point on the screen. Only one touch can be registered on the Brain display. `Touch` fields:
  - pos: [Point](#point)
  - event: [TouchEvent](#touchevent)
- `ControllerUpdate`: TBD
- `USD`: Mount or unmount a directory as the V5's SD Card. Robot programs will be able to read and write to this directory. Fields:
  - root: string (path to sd card's root), or null to unmount
- `VEXLinkOpened`: The VEXLink server has successfully opened a connection. Fields:
  - port: [SmartPort](#smartport).
  - mode: [LinkMode](#linkmode)
- `VEXLinkClosed`: The VEXLink server has closed a VEXLink connection. Fields:
  - port: [SmartPort](#smartport).
- `CompetitionMode`: Update the competition mode. Fields:
  - connected: boolean
  - mode: [CompMode](#compmode)
  - is_competition: boolean
- `ConfigureDevice`: Configure a port as a specified device. Fields:
  - port: [Port](#port)
  - device: TBD
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

* `Fill`: Fill a shape on the screen.  Fields:
  * shape: [Shape](#shape)
* `Stroke`: Draw the outline of a shape on the screen. Fields:
  * shape: [Shape](#shape)
* `CopyBuffer`: Draw a pixel buffer to the screen with a given stride and start and ending coordinates. `CopyBuffer` fields:
  * top_left: [Point](#point)
  * bottom_right: [Point](#point)
  * stride: nonzero integer
  * buffer: [``Color``](#color) array.

### Shape

An enum that describes a shape to be drawn on the screen.
Shape has these variants:

* `Rectangle`: Rectangles are drawn starting at the top left coordinate and extending to the bottom right coordinate. `Rectangle` fields:
  * top_left: [Point](#point)
  * bottom_right: [Point](#point)
* `Circle`: `Circle`s are drawn with a coordinate at the center of the circle and a radius. This variant has these fields:
  * center: [Point](#point)
  * radius: integer
* `Pixel`: Draw a single pixel at a coordinate. `Pixel` fields:
  * pos: [Point](#point)

### Point

A struct that stores a pixel coordinate.
The origin is at the top left of the screen.
``Point`` fields:
- x: integer
- y: integer

### DeviceStatus

WIP

An enum representing the state of a device.
Every time the state of a device on any port changes, this enum will be sent.
``DeviceStatus`` variants:
- ``Motor``: The state of the motor has changed.
  ``Motor`` fields:
  - velocity: float in radians per second.
  - reversed: boolean
  - power_draw: float in Watts
  - torque_output: float in Nm
  - flags: [``MotorFlags``](#motorflags)
  - position: float in radians
  - target_position: float in radians
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

## TouchEvent

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

### Battery

A struct containing information about the current state of the battery
Fields:

- `voltage`: The output voltage of the battery. Float value in Volts.
- `current`: The current draw of the battery. Float value in Amps.
- `capacity`: The remaining energy capacity of the battery. Float value in Watt-Hours.

## Notes

- Rust implementation available at <https://docs.rs/pros-simulator-interface>.
- `pros-simulator-server` only uses Stdio to communicate, but other streams are possible.
- Data over Stderr should be considered unstructured logs and may be shown to the user. As such, if the code executor crashes, its error message will be visible.
