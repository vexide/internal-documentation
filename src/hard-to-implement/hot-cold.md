# Hot/Cold Linking

(Since this page was written, we've added an equivalent to hot/cold linking to vexide called Differential Uploading.)

Hot/Cold linking is a feature of VEXos that allows for a user program to be split into two files, named the hot and cold binaries.
The purpose of splitting your binary up into two files
is that the code in the libraries you use, PROS, pros-rs, vexide, VEXCode, etc.
can be uploaded once in the cold binary and never again.
User code ends up in the hot binary.
The hot binary is the only binary that is uploaded when you make changes.
When working with a large project, hot/cold linking can save a huge amount of time spent uploading.
A PROS V3 project can take over 30 seconds longer to upload when using a monolith binary.
Currently vexide only supports monolith style linking
which puts all code into one binary.

<!-- TODO: the hot/cold boot process, possible implementations, and why its a priority -->
