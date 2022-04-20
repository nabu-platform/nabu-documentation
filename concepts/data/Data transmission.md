# Data transmission

Once we have marshalled our data in a way other systems understand, we need to actually get that data to those systems so they can get started on the unmarshalling process.
Transferring data from one system to another is done via a "protocol" which describes the structure and sequence of messages that the systems must exchange in order to transmit the data.

binary
textual

request/response like http
both directions like websockets


Even though all data is fundamentally binary (existing of 0's and 1's) because that is all the computer knows, we still differentiate between binary and textual formats. The difference is whether people can read the end result as well, which we dub as textual. 