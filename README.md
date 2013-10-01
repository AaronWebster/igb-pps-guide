'igb-pps' test system installation guide
========================================

### v.1 - written by Balint Ferencz (ferencz@mit.bme.hu)

Introduction
------------

This document serves as a quick guide to install and test the Intel i210
Ethernet adapter based clock synchronization solution. It fills the missing gaps
in my master's thesis. The thesis in pdf format is also available from the
location of this document ([my bitbucket account][1]).

The document assumes that the reader has a Ubuntu 12.04 LTS installed on some
sort of virtual or non-virtualized environment. The installation and setup of
the OS or the virtualisation system is not covered in this document. There are a
few constraints in the selection of the underlying system as it needs:

1.  a Linux kernel with version 3.0 or greater,

2.  and a properly installed i210 adapter is also required.

To follow the steps of the guide, one needs to login into the selected and
installed OS.

[1]: <http://bitbucket.org/fernya>

Preparation
-----------

On Ubuntu Linux the following packages should be installed for kernel
compilation:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo apt-get install build-essential fakeroot devscripts \
crash kexec-tools makedumpfile kernel-wedge git libncurses5\
libncurses5-dev libnewt-dev
$ sudo build-dep linux-image-$(uname -r)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `build-essential` package brings the most important building tools - with
the kernel headers it's enough for the compilation of the igb driver itself, the
other packages are needed for the kernel compilation, as the PTP subsystem is
disabled in the most of default kernels. The recompilation of the developer
maintained kernel tree on Ubuntu Linux is covered by the following link:
[http://blog.avirtualhome.com/compile-mainline-kernel-ubuntu/][2]  
For other systems the Google has many other relevant results. However, the most
important step in the kernel configuration on every OS is to enable the PTP
subsystem in the source (e.g. `CONFIG_PTP_1588_CLOCK=m` option in the 3.2.0
kernel configuration file - **it has dependencies**).

For the Ubuntu users' convenience I have compiled and packaged a PTP enabled
AMD64 compatible version of the kernel. It's available on the [previously
mentioned][3] bitbucket site. The compiled kernel image and header files should
be installed and and the system should be rebooted with the new kernel.

**Note:** my provided kernel version (as it's based on 3.2.0), my shipped driver
and utilities do not include the TXSTAMP_ONE_STEP and the auto PHC discovery
functions!

[3]: <https://bitbucket.org/fernya>

Getting and setting up the software
-----------------------------------

The driver is available in the* igb-pps*[ branch on my bitbucket site][4]. To
obtain it you can download it as a tarball, or you can check it out with `git`
which is preferred.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ git clone https://fernya@bitbucket.org/fernya/igb-pps.git
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The repository contains three parts, the driver itself, and two utilities.

### igb-pps driver

This driver is a fully functioning igb driver (based on version 4.0.17) with the
necessary additions to enable the advanced functionality on the adapters. To
compile and load it you may issue the following commands:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ cd igb-pps
$ cd src
$ make
...
$ sudo make install         # if you want modprobe the driver (preferred)
...
$ sudo rmmod igb            # removes the previously loaded stock driver
$ sudo modinfo igb
filename:       /lib/modules/3.2.0-23-ptp/kernel/drivers/net/igb/igb.ko
version:        4.0.17-PTP
license:        GPL
description:    Intel(R) Gigabit Ethernet Network Driver
...
$ sudo modprobe igb         # loads the modified driver
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The PTP postfix denotes that our driver is installed to the proper place
therefore the `modprobe `command  can insert the proper driver into the kernel.

### perpps

The `perpps `utility controls the nPPS frequency output on the card. It is
located in the perpps directory of the igb-pps package. The utility needs proper
privileges to carry out its operations, so without any further configuration it
only works as *root*. The provided source code may be compiled by issuing the
`make` command.

The two most common use cases are the enabling of the PPS signal and enabling of
a higher frequency signal. The output signals have 50% duty cycle on the i210
adapters. The maximum output period is constrained by sanity reasons on i210.
The cards could output at most T=125 ms signals with the usage of the -P switch.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo ./perpps -d /dev/ptp0 -p 1            # Enables PPS HW output on
$ sudo ./perpps -d /dev/ptp0 -P 0,1000       # Generates 1MHz output on ch0
$ sudo ./perpps -d /dev/ptp0 -P 0,1000000000 # Fails with 1PPS
PTP_PEROUT_REQUEST failed: Invalid argument
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This program duplicates the functionality of the sysfs entries of the PTP-API,
especially those which are controlled by the '`enable_pps`' and '`period`'
files. The user should decide which way he or she wants to control the output
signals.

### extts

To improve the absolute accuracy of the clock sync system, the external time
stamping feature of the adapter is used to clone an external reference clock.
Majority of the reference clocks provide two outputs, the first emits the exact
phase and frequency information (e.g. 1PPS pulse), and the second emits an
absolute time string (e.g. serial output). In my thesis I've used a GPS receiver
as reference time source, therefore I've enabled this program to clone both the
1PPS signal and the serial time string output. To achieve this functionality
I've relied on the gpsd library, which is needed to compile the `extts
`software.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo apt-get install libgpsd-dev
$ cd <insert path here>/igb-pps/ts2phc
$ make
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By issuing these commands the `ts2phc` is ready to run. The program supports the
following functions:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ ./ts2phc
$
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Several examples are provided in the latter sections of this document.

### linuxptp

The cards do not implement the state machine of the IEEE 1588 synchronization
protocol, therefore we need a software stack to do the synchronization itself.
The author of the PHC-API in the kernel (Richard Cochran) also develops a
fully-fledged implementation of the IEEE 1588 synchronization daemon called
`linuxptp`. It can be obtained from [here][5], in this guide I used the 1.3
release of the code.

[4]: <https://bitbucket.org/fernya/igb-pps/overview>

[5]: <http://linuxptp.sf.net>

Setting up the physical measurement environment
-----------------------------------------------

The controllers have four software definable pins per Ethernet port which can be
used as a CMOS level IO ports. These IO pins can show or influence the internal
clock functions of the controllers. More precise information is available in the
[respective datasheets][6].

[6]: <http://ark.intel.com>

**WARNING:** Never connect 5V signals on the inputs!  
**WARNING:** Watch out for stray currents in the ground path!

### Physical layout of the i210

The i210 cards have their SDP pins connected to a header, which supports the
connection of 3 independent signals. The layout is simple, the three pins closer
to the RJ45 jack are the ground pins, and in the other column are the SDP0-2
signals from top to bottom. By default the topmost functions as input (SDP0) and
the one below that as output (SDP1).



### Checking the output

The most simple measurement is the checking the frequency of the adapter. This
measurement consists of the following parts:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PC ----> Ethernet adapter ----> Logic analyzer or oscilloscope
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The default output is configured to the SDP1 therefore the sink has to be
connected to that pin. Don't forget the proper ground contact and shielding! By
the issuing the following commands the 1PPS output is enabled:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo ./perpps -d /dev/ptp0 -p 1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now the card should emit ~1Hz frequency square wave with 50% duty cycle on its
SDP1 output. On the kernel message logs we can also see some PPS events (only if
we included the PPS debugging messages feature into the kernel). If we issue the
command:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo ./perpps -d /dev/ptp0 -P 0,1000
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

the card should emit ~1MHz frequency square wave.

**WARNING:** To ensure the proper phase of the clock output these commands
should be issued only in synchronized state of the adapters! The first rising
edge of the signal is always aligned to the start of the second with respect to
the adapters onboard clock.

### Checking the input



### Setting up a reference signal relay

In my master thesis I've demonstrated a reference signal relay for IEEE 1588
networks. This measurement setup uses both functionality of the adapters.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                         +---> IEEE 1588 LAN
                         |
REFCLK --- 1PPS ---> Ethernet adapter --- 1PPS ---> Scope CH1
|       |                |
|       `-> Scope CH0    `--+------+
|                           |      |
`----- serial line ----------> PC <-------> extts, linuxptp

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


