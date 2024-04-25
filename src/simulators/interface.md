# Vexide Simulator Interface

Vexide Simulator Interface is a protocol that allows simulator code executors and frontends to communicate. Software that implements it can work interchangeably, similarly to the Language Server Protocol.

## Communication

The code executor and frontend talk over a stream in [newline-delimited JSON format](https://jsonlines.org/).

The code executor sends events that happen inside the simulator that the frontend might want to know about. Use this to monitor robot code progress, simulated LCD updates, log messages, and more.

The frontend sends messages to the code executor to control the robot code environment, simulating changes in robot hardware (like controller input and LCD touch events) or competition phase.

## Events

Events are sent from the simulator backend to the frontend to describe simulator state changes.

- `ScreenDraw`: Draw something on the screen. `ScreenDraw` fields:
  - command: [DrawCommand](#drawcommand)
  - color: integer
- ScreenClear: Fill the entire screen with one color. `ScreenClear` fields:
  - color: integer
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
- `USD`: TBD
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

## Data types

Several different enums and structs are used for communication to and from the backend.

All enum datatypes are encoded using externally tagged representation; see [Serde enum-representations](https://serde.rs/enum-representations.html).
Colors are encoded in 0rgb8 format. 

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
  * buffer: 32 bit integer array of colors.

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
A point stores two integers;
one for the x coordinate and one for the y coordinate.

### DeviceStatus

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

## Notes

- API documentation available at <https://docs.rs/pros-simulator-interface>.
- `pros-simulator-server` only uses Stdio to communicate, but other streams are possible.
- Data over Stderr should be considered unstructured logs and may be shown to the user. As such, if the code executor crashes, its error message will be visible.
