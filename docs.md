# Using Microcontrollers in Scientific Research

## Techniques and examples to achieve

* Datalogging and Remote Control
* Fast prototyping
* LTE networking with Blues Wireless
* Low-cost adaptable system

## Abstract

lksfglkjs

## Introduction - Is this for Me?

This guide documents techniques used to capture sensor data and remotely control
an electronic system. It is particularly focused on scientific use cases, and
should be of interest to researchers looking to implement similar systems.

The motivation may be one or more of:

* Continuous monitoring of a physical process
  * e.g. temperature, pressure, light, sound, image, motion, chemical
* Controlling an output transducer
  * e.g. motor, pump, valve, light, relay, LCD screen, alarm
* Remote operation
  * Deployment in difficult to access locations
  * Networking for real-time data

Every application is different, so depending on the requirements (including
time/budget/scale/complexity), there may be suitable off-the-shelf systems
already available. It is worth taking some time to evaluate whether the
requirements can be met by (or adapted to) an existing system, as there could be
significant savings in development time and ongoing support.

Assuming a complete off-the-shelf system is not available, a custom solution is
required. However, custom design is challenging so it is important to build from
existing components wherever that makes sense. This concept has been used to
inform design choices throughout this guide. For example, the choice to use
hardware and software targeted at the hobbyist market.

The approach described here particularly suits applications with the following
characteristics (most important first),

* High flexibility due to evolving/unknown requirements
* Fast development time
* Low volume, in the order of 1 - 10 units
* Individual or small team development
* Cost sensitive (compared to industrial process control equipment)
* Accessible to non domain experts
* Low power consumption

Caveats

* Very high reliability tends to be in conflict with quickly developed
  prototypes. Something simpler would be better suited to e.g. safety critical
  applications.
* There can be a learning curve, particularly for those unfamiliar with
  software/electronics engineering.

The techniques have been selected intentionally to not require a high degree of
specialist knowledge. Where technical knowledge is required, it is well
documented online, so a motivated researcher should be able to pick it up.

## Architecture - Anatomy of a system

This chapter breaks down a typical system into key components. It discusses the
design options a developer might evaluate, leading to recommended choices for an
example system.

### Microcontroller

The system revolves around a microcontroller (MCU) board. Some will know this as
"an Arduino", but Arduino is just one of the available ecosystems. It's
essentially a small computer, and much of the programming work will be writing
code to run on this. Assume that the sensors and output transducers all connect
into the microcontroller.

There are a huge variety of cost optimised MCUs, but for low volume applications
it makes sense to choose based on ease of development. Newer MCUs tend to have
more than enough RAM, Flash Storage, peripherals and clock speed, so software support
is likely to be more critical than the exact hardware specs.

The combination of availability, community support, and capability make the
following families of microcontrollers a good starting place:

| Family | Defining feature          | Example Board                                                                |
| :----- | :------------------------ | :--------------------------------------------------------------------------- |
| ESP32  | Built in Wifi / Bluetooth | [UM Feather S3](https://esp32s3.com/feathers3.html)                          |
| STM32  | Low power                 | [Blues Wireless Swan](https://blues.io/products/swan/)                       |
| RP2040 | Low cost                  | [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/) |

They all enable programming frameworks like Arduino and Circuitpython, which are
discussed in the Language section. There are some networking differences
detailed in the Networking section

Board examples have been provided, but other form factors are available, e.g.
including a display, exposing more pin connections or with physical
compatibility with particular accessory boards that plug on top (sometimes known
as shields, HATs or featherwings)

TODO - photos of boards, shields

A small linux computer like a
[Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
is also a valid choice for many applications, particularly if dealing with
larger volumes of data associated with audio or video, or for connecting to a
device with linux driver support. Networking capabilities are also much more
advanced with linux, e.g. remote login with SSH. The main downsides would be
cost, complexity, power consumption and latency (i.e. unsuitable for controlling
hardware that needs precise microsecond timing).

### Language

Microcontrollers can be programmed in various ways, but the fastest way to make
progress is to build from an existing framework like Arduino or Circuitpython.
These each have their merits, and which one to choose is a subjective topic.

For scientific automation, Circuitpython is a strong candidate. Typically the
board is connected to a PC and shows up as a USB drive. The user's code exists
as a plain text file that will run immediately after being edited. No slow code
compilation step is needed because the MCU is running a python interpreter to
read the file directly. Having a short developer iteration time is ideal for
prototyping and learning.

On the other hand, Arduino (which is based on C/C++) uses a compiler and can
take in the order of 30s to flash any code changes to the board. This will slow
development time, but this more traditional paradigm gives Arduino code an
advantage in terms of better access to all hardware functions (like interrupts),
and execution speed (e.g. where millisecond scale timing matters).

Perhaps the most critical factor is library availability (i.e. software to
interface easily with sensor/accessory boards, or extended software features) and
documentation. Arduino and Circuitpython both excel in this area. For those
looking to quickly integrate existing breakout/expansion boards, sensors,
displays etc, having a library already written and tested will greatly reduce
development time. Circuitpython is supported by Adafruit, a company that
specialises in hobbyist electronics and sells a wide selection of compatible
boards, all with Circuitpython drivers and "Learn Guides" for getting started.

Arduino has been established for longer so has even more libraries, with the
majority being well supported and documented. However, certain tasks may be
better suited to a particular language, for example an algorithm manipulating
memory at the bits / bytes level would be well served by Arduino, but parsing
arbitrary length strings of text/JSON (e.g. from a web API) is easier in python.
Circuitpython has good support for simple graphical user interfaces or displays
too.

Scientists may find python preferable, as it is one of the most versatile
high-level languages for data processing, visualisation, machine learning etc.
It is comparable to Matlab or R.

Micropython is similar to Circuitpython, but with fewer supported libraries and
MCU boards. It may suit users looking for extra hardware functionality like
threading and interrupts, while still being an interpreted language. For most
users, the set of tradeoffs provided by Circuitpython or Arduino are more well
rounded.

Similarly, for advanced projects there are other options ranging from "bare metal"
Embedded C, to Real Time Operating Systems (RTOS) like Zephyr or FreeRTOS. These
tend to suit professional product development on a larger scale and have too
much of a learning overhead to be considered here.

Summary table

|                          | Arduino | Circuitpython |
| :----------------------- | :-----: | :-----------: |
| Software iteration speed | &#9744; |    &#9745;    |
| Libraries and support    | &#9745; |    &#9745;    |
| Low level MCU features   | &#9745; |    &#9744;    |
| Execution speed          | &#9745; |    &#9744;    |
| Language versatility     | &#9744; |    &#9745;    |

As always, choose whichever has the best support for the application. The MCUs
recommended in this guide can support either of these frameworks, so it is
possible to change part way through a project if necessary.

### Networking - keeping out of the weeds

There are many pitfalls with getting MCUs connected to the internet,
particularly if they need to run unattended for extended periods of time.

While wifi has become ubiquitous, there are inherent difficulties, e.g
institutional wifi networks like eduroam may require certificates, and saving an
individual's username and password in the MCU code is bad practice for security
and because it may change.

Similarly, mobile networking with 4G/LTE typically requires SIM cards, monthly
payments and choosing a suitable provider with the right coverage.

The requirement to have code to sensibly handle any errors and manage
reconnections compounds the problem.

These issues can be largely avoided by using a specialist networking card like a
[Blues Wireless Notecard](https://blues.io/). This fits the theme of using
existing solutions where possible. It uses mobile data, although a wifi version
is available too.

Notecard Pros

* Data plan and integrated SIM all included in the hardware price. No separate
  contracts to manage.
  
* Notecards uses their own independent MCU to manage the connection and queue
  data. This means the primary "host" MCU doesn't need any networking software
  and is free to manage the rest of the system. This can greatly help with
  timing and responsiveness.

* The Blues Notehub dashboard enables administration of multiple devices.
  Notecards connect to this via a VPN so they are not visible to the open
  internet, which is ideal for security.

* A hierarchy of "environment variables" allows custom settings for
  different devices in a fleet. e.g. to adjust a target temperatures in a
  heating system.

* An "Outboard Device Firmware Update" feature allows the host MCU to be
  easily reprogrammed remotely. The Notecard acts like an attached PC to flash
  the MCU. This greatly reduces the complexity of remote firmware updates and
  the risk of rendering the system inoperable.

* Designed for low power operation, requires almost no current (8uA) during
  periods between network synchronisations.

* Technical support is readily available.

Notecard Cons

* Notehub doesn't offer any plotting facilities, data must be forwarded
  ("routed") to another service for longer term storage and visualisation.

* It is best suited to applications with low data requirements, e.g. sending
  kilobytes per day, not megabytes. So it doesn't handle audio, video or
  images well.

* Communication between the host MCU and the Notecard typically uses the I2C
  protocol. Certain operations can be relatively slow (~100 milliseconds) which
  can impact the responsiveness of the host MCU.

* Mobile network coverage is required, although it is possible to use
  external directional antennas to increase the range.

Alternatives

If higher data rates are needed, consider a 4G-to-wifi router with a standard
SIM and appropriate data plan. e.g.
[TP-Link TL-MR6400](https://www.amazon.co.uk/TL-MR6400-Unlocked-Configuration-Required-External/dp/B016ZWXYXG/).
A single board linux computer (e.g. Raspberry Pi) may be appropriate in this
case, and would save having to manage the network on an MCU.

For locations without sufficient network coverage, look at options from e.g.
[Ubiquiti](https://uisp.com/uisp-overview), which can extend wifi networks
\>10km.

For low data rates and low power, LoRa (LongRange) networking. Blues Wireless
have a LoRa solution called
[Sparrow](https://blues.io/products/sparrow-iot-sensor-clusters-over-lora/) that
integrates with their notecards. Some locations may already have LoRa networks
in place that permit new nodes to join.
[The Things Network](https://www.thethingsnetwork.org/) has more information.

Arduino offer their own [cloud](https://cloud.arduino.cc/) service. It is likely
the fastest and easiest way to get going for projects using Arduino branded
boards and arduino software. It provides connection management, security, Over
The Air updates and data dashboards. It doesn't currently have support for
4G/LTE networks, and the walled-garden approach may become limiting, e.g. if
looking to scale to multiple devices. However, for smaller WiFi based projects
this has a lot going for it.


Example Project

Having reviewed the options for MCU board, language, and networking it is
possible to make some selections for the example project.

* Microcontroller - Blues Wireless Swan (STM32 based)
* Language - Circuitpython
* Networking - 4G/LTE via Blues Wireless Notecard

The Swan microcontroller board from Blues Wireless is compelling due to:

* Low power design
* Supporting the outboard DFU feature without any hardware modifications
* The Feather form factor and Qwiic connection ports make it easy to extend
  the functionality by connecting breakout boards or Featherwings.



### Inputs

### Output

### Server Software

* Blues Wireless Notehub

* Initial State

* Github Pages

### Firmware updates

### Breakout Boards

Many hardware components are readily available in small inexpensive circuit
"breakout" boards that are often associated with "hobbyist" electronics. They
are easy to use and are often designed to work together. These boards would not
typically be used in professional products, but that is down to economies of
scale. The semicondictor chips on the breakout boards are high quality
up-to-date parts.

PHOTO



## Comparison with

Scada, PLC (ladder logic), Arduino, NI PXE

## Methods

## Beyond the basics

Circup
Git

## Results

## Discussion

## Help and References

* Circuitpython
  * Introduction Learn Guide <https://learn.adafruit.com/welcome-to-circuitpython/overview>

  * Main Documentation <https://docs.circuitpython.org/en/latest/README.html>

  * CircUp <https://docs.circuitpython.org/projects/circup/en/latest/index.html>

* Blues Wireless System
  * Quickstart and API documentation <https://dev.blues.io/>

  * Help Forum <https://discuss.blues.io/>

* Initial State (Cloud Dashboard)

  * <https://www.initialstate.com/>
