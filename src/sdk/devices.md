# Device APIs

This chapter documents some additional details about how VEXos interfaces with devices.

## Distance Sensor

The VEX V5 Distance Sensor is known to internally use the [ams-OSRAM TMF8805] Time-of-flight sensor.

[ams-OSRAM TMF8805]: https://look.ams-osram.com/m/1198e66f7a81b20c/original/TMF8805-Time-of-flight-sensor.pdf

The SDK exposes a status code for connected distance sensors via `vexDistanceStatusGet`. While VEX [documents this code](https://www.vexforum.com/t/v5-distance-sensor-status-code-difference/147191/2) as having "no customer use," it may be possible use it to determine whether a distance sensor is working properly since a healthy sensor will always use the status `0x82` or `0x86`.
