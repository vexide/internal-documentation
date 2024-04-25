# Project info

vexide, our namesake library, is the successor to pros-rs.
Instead of linking with and creating bindings for a library like PROS, vexide implements everything on its own.
Every line of code in a vexide program is open source and written in Rust.

The major technical difference between vexide and pros-rs is that vexide directly calls VEXos jumptable functions without going through ``libv5rt``'s wrapper functions.
This allows us to implement everything ourself instead of linking to ``libpros``.
vexide is also different from other Brain libraries in that it doesn't use an RTOS. vexide uses Rust async/await for multitasking.
vexide programs are incredibly small for not being hot/cold linked and have higher stack and heap sizes than any other library.
