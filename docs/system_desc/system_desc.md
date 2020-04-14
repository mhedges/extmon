System Description 
==================

Overview
--------

The ExtMon hardware and electronics are detailed in \[FIXME:ref\]. To
summarize, an individual channel corresponds to a single FE-I4b pixel
chip paired with a silicon sensor. Channels are powered and monitored
through Pixel Interface Boards (PIBs) and the data acquisition (DAQ)
from the pixels is done through MicroTCA electronics. A PIB is a custom
printed circuit board consisting of power distribution and monitoring
hardware for communication and control for up to six channels through a
BeagleBone Black single-board ARM-based computer. More detailed
documentation of the PIB and associated components can be found in
[docdb-12671](https://mu2e-docdb.fnal.gov/cgi-bin/private/RetrieveFile?docid=12671&filename=Mu2e_PIB_v1.52.docx&version=11).
The details of the DAQ via MicroTCA are detailed in \[FIXME:ref\].

The entire ExtMon will consist of 20 channels, which will require a
total of four PIBs. Each channel is controlled through its PIB via
hardware controllers on the BBB. The BBB has a network interface that
can connect to a standard ethernet switch, allowing for multiple PIBs to
be controlled and monitored remotely. Separately from channels, the
event trigger for the ExtMon system consists of several plastic
scintillators with photomultiplier tubes (PMTs). These PMTs require high
voltage, which is provided by a VME crate with a network interface
through the Simple Network Management Protocol (SNMP).

Network Configuration
---------------------

All physical device and sensor controlls are accessible through standard
networking protocols. This allows for a simple subnetwork structure with
all devices connected to a switch, with one machine dedicated as the
subnetwork gateway (GTWY). A diagram of this configuration is shown in
Figure 1.

![Network configuration for ExtMon device controllers. Solid lines
denote network connections and dashed lines data transfer
connections.](../img/network.png "network")
