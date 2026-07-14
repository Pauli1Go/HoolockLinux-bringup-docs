# Touch support for the iPad 7 under Linux
This bring-up was developed with extensive assistance from OpenAI Codex. Codex helped reverse engineer the Apple SPI/Z2/HBPP14 touch protocol, analyze firmware and iOS behavior, compare it with Linux, and develop the resulting kernel drivers with a combined active time of ~11 days.

## Overview

This document describes the final Linux touch implementation for the seventh-
generation iPad, specifically the cellular `iPad7,12` (`J172`) based on the
Apple A10 / T8010 SoC.

The finished implementation is a normal in-kernel input stack. It powers and
initializes the touch controller during boot, loads the required firmware,
registers `/dev/input/event0`, and reports single- and multi-touch input through
the standard Linux input subsystem. No userspace touch daemon, manual driver
rebind, module-parameter setup, or display contact during probe is required.

The final tested system provides:

- automatic touch initialization during a fresh boot;
- a standard `apple-z2` SPI touchscreen device;
- single-touch and simultaneous two-finger input;
- correctly scaled 1620 x 2160 coordinates;
- 10 units/mm axis resolution, reported by libinput as a 162 x 216 mm panel;
- stable operation on the graphical Linux desktop; and
- an internal NVMe root filesystem mounted read/write at the same time.

## Supported hardware

The implementation and validation described here target:

| Component | Value |
| --- | --- |
| Product | iPad 7 Cellular |
| Apple model | `iPad7,12` |
| Board identifier | `J172` |
| SoC | Apple A10 / T8010 |
| Touch transport | Apple Z2 over SPI3 |
| Touch controller family | BCM15900B0-based Apple multitouch assembly |
| Linux input device | `iPad7,12 Touchscreen` |
| Logical display size | 1620 x 2160 |

Other Apple Z2 devices still use the generic portions of `apple_z2.c`, but the
J172 power-up, HBPP14 firmware path, report setup, and Device Tree description
are explicitly selected by the `apple,j172-touchscreen` compatible.

## How the implementation was developed

The driver was developed by comparing several independent sources rather than
copying a single existing implementation:

- the J172 Apple Device Tree (ADT), which provided SPI, GPIO, interrupt, clock,
  power, and firmware metadata;
- the matching multitouch firmware container and its HBPP14 records;
- the iOS `AppleSamsungSPI`, `AppleHIDTransportDeviceSPI`,
  `AppleHIDTransportProtocolZ2`, and multitouch driver behavior;
- passive iOS RAM-only oracles that captured already-completed protocol data
  without adding extra SPI transactions;
- the Project Sandcastle implementation as a hardware-behavior reference;
- passive T8010 PMGR and PLL register readings;
- Linux SPI, DMA, IRQ, and evdev captures; and
- repeated fresh-boot tests with strict stopping rules at the first failed
  initialization gate.

This comparison established the exact controller setup and command order that
the J172 touch device expects. The important result was not a single magic
command, but a complete and correctly ordered lifecycle: power and reset,
SPI configuration, controller readiness, HBPP14 firmware upload through
SmartIO DMA, two firmware-ready notifications, Z2 device discovery, report
initialization, and finally runtime interrupt handling.

The implementation was then cleaned up so that protocol knowledge lives in
the touchscreen driver, while the SPI and DMA drivers remain reusable
controller drivers.

## Final kernel architecture

The support is split into the normal Linux subsystems described below.

### T8010 touch clock

`drivers/clk/clk-apple-t8010-touch.c` implements the dedicated T8010
touch-clock gate and divider exposed by PMGR.

The driver:

- enables and disables the 24 MHz touch clock used by the touch assembly;
- waits for the PMGR busy bit to clear;
- preserves the required atomic disable behavior; and
- handles the full ten-bit divider encoding, including logical divider 1024
  being represented by register value zero.

This 24 MHz touch clock is separate from the SPI3 controller input clock.

### Apple SmartIO and SIO DMA

The T8010 uses the Apple SmartIO coprocessor for the large transmit DMA
operations required by the firmware upload.

The implementation consists of:

- `drivers/soc/apple/apple-sio-core.c`, which boots and manages SmartIO;
- `drivers/dma/apple-sio-dma.c`, which exposes SIO channels through the Linux
  DMAengine API; and
- `include/linux/soc/apple/sio.h`, which contains the shared kernel interface.

The common T8010 Device Tree node describes the SmartIO register regions,
power domain, firmware, four hardware interrupts, and 64 DMA channels. SPI3
uses SIO transmit channel `0x1a` for the firmware payload.

The SIO DMA driver is transport infrastructure. It does not inspect Z2 or
HBPP14 payload contents.

### T8010 SPI controller support

`drivers/spi/spi-apple.c` contains the generic T8010 SPI controller support.
It knows about controller registers, FIFO behavior, chip select, PIO,
DMAengine transfers, word sizes, timing, and the legacy T8010 descriptor
format. It deliberately contains no J172, Z2, HBPP14, or firmware-payload
detection.

The T8010 path uses `transfer_one()`, leaving message sequencing, chip-select
handling, transfer delays, `actual_length`, DMA mapping, and DMA cache
synchronization to the SPI core.

Transfers are selected from generic properties:

- short transfers use the T8010 PIO path;
- suitable transmit-only transfers use DMAengine when an SG mapping exists;
- unsupported or unmapped DMA shapes fall back to PIO; and
- 8-bit and 16-bit word modes are selected from the SPI transfer itself.

Clock division is calculated from `speed_hz` and the actual controller input
clock. J172's physical SPI3 `nclk` was derived from the T8010 clock tree and
measured PMGR/PLL state as 55,384,615 Hz. With a requested maximum speed of
24 MHz, the generic calculation selects divider 3, producing an effective SPI
clock of approximately 18.46 MHz. No fixed J172 divider is embedded in the
controller driver.

### Apple Z2 touchscreen driver

`drivers/input/touchscreen/apple_z2.c` owns the complete touch protocol.

For J172 it handles:

- GPIO-controlled analog and LDO power;
- reset and display-sync signals;
- the pre-reset SPI setup transfer;
- the initial readiness transaction;
- parsing and uploading the HBPP14 firmware records;
- large firmware transfers through the SPI/SIO DMA path;
- both firmware-ready interrupt waits;
- Z2 command framing, checksums, and command/reply validation;
- DeviceInfo discovery;
- surface descriptor query (`D9`);
- report metadata and report initialization;
- runtime IRQ handling; and
- conversion of multitouch records into Linux input events.

The driver contains an explicit protocol state machine:

```text
OFF
  -> POWERED
  -> SPI_CONFIGURED
  -> BOOT_IRQ
  -> HBPP_READY
  -> FIRMWARE_READY
  -> DEVICE_INFO
  -> RUNTIME
  -> REPORTS_READY
```

State is advanced only after a successful transfer or a validated reply.
Boot errors, retries, and shutdown reset protocol state, runtime parity, and
surface information. This prevents a failed firmware attempt from leaving the
next probe in a partially initialized state.

The two firmware-ready waits intentionally do not reinitialize the completion
between waits. If the second IRQ arrives before the first wait has fully
returned, its completion token is retained instead of being lost.

J172-only state transitions are guarded by the J172 device selection, so
existing non-J172 Z2 devices do not depend on a J172-specific predecessor
state.

### Device Tree description

The Device Tree connects all components without runtime parameter injection.

The J172 touchscreen node describes:

- SPI3 mode 3 and a maximum requested speed of 24 MHz;
- the measured 55,384,615 Hz SPI3 controller input clock;
- external chip select on AP GPIO85;
- SPI pins GPIO82-84;
- touch IRQ on AP GPIO21, falling edge;
- LDO power on GPIO29;
- active-low reset on GPIO31;
- analog power on GPIO145;
- display synchronization on GPIO176 and GPIO177;
- the dedicated T8010 touch clock;
- SIO transmit DMA channel `0x1a`;
- firmware name `apple/dfrmtfw-j172-k1f19-6.bin`; and
- a 1620 x 2160 logical touchscreen size.

The SmartIO node and its four SoC interrupts are in the common T8010 Device
Tree because they describe SoC wiring rather than J172 board wiring.

## Required firmware

The driver uses Linux `request_firmware()` and requires two firmware files:

```text
apple/t8010-smartio.bin
apple/dfrmtfw-j172-k1f19-6.bin
```

The first file boots the T8010 SmartIO coprocessor. The second is the J172
multitouch firmware container used by the HBPP14 loader.

These Apple firmware files are not part of the kernel source. They must be
obtained legally from firmware for the matching device and placed in the
root filesystem or early initramfs under the paths above. For early automatic
probe, both files should be available before the real root filesystem is
mounted. If the Z2 firmware is absent from the initramfs, `request_firmware()`
can delay probe for roughly a minute and touch initialization will miss the
boot sequence.

## Boot requirements

The tested Linux system boots through checkm8/palera1n, PongoOS, and m1n1.
The patched [m1n1-ipad7](https://github.com/Pauli1Go/m1n1-ipad7) fork is
required for the current complete system because it populates the T8010 NVMe
Host Memory Buffer region in the Linux Device Tree. Without that loader
change, the internal NVMe root filesystem does not initialize, even though
the touch driver itself is independent of NVMe.

The working boot chain is:

```text
DFU/checkm8
  -> PongoOS
  -> patched m1n1
  -> Linux Image + J172 DTB + firmware-containing initramfs
  -> internal NVMe root filesystem
  -> graphical Linux desktop
```

## Automatic touch initialization

During a normal fresh boot, the kernel performs the following sequence:

```text
Power and clocks
  -> reset and SPI setup
  -> first valid readiness reply
  -> HBPP14 firmware upload through SmartIO DMA
  -> first firmware-ready IRQ
  -> second firmware-ready IRQ
  -> DeviceInfo
  -> D9 surface descriptor
  -> report discovery and initialization
  -> contact-free completion of probe
  -> runtime IRQ enabled
  -> /dev/input/event0 registered
```

The probe must complete without requiring a finger on the display. Once the
runtime IRQ is enabled, touches are parsed and emitted through evdev.

The validated controller information was:

```text
DeviceInfo: platform 34, interface 17, maximum packet 5891
Surface:    15552 x 20736 controller units
Bounds:     x = -223..15329, y = -222..20513
Input:      1620 x 2160 pixels, resolution 10 x 10 units/mm
```

The driver scales controller coordinates into display pixel coordinates and
uses standard multitouch slots, tracking IDs, `BTN_TOUCH`, and synchronization
events. Desktop input therefore works through ordinary evdev/libinput.

## Kernel configuration

The relevant built-in configuration is:

```text
CONFIG_INPUT_TOUCHSCREEN=y
CONFIG_TOUCHSCREEN_APPLE_Z2=y
CONFIG_SPI_APPLE=y
CONFIG_DMADEVICES=y
CONFIG_APPLE_SIO_DMA=y
CONFIG_COMMON_CLK_APPLE_T8010_TOUCH=y
```

Building these drivers into the kernel avoids module and firmware-ordering
problems during the early boot sequence.

## Validation

The final cleaned implementation was tested from a fresh boot on the current
HoolockLinux iPad 7 development base.

Before human input, the following conditions were verified:

- patched m1n1 populated the 18 MiB NVMe HMB region;
- the Apple PCIe and NVMe endpoints were present;
- `nvme0` reported `live`;
- `/dev/nvme0n1p2` was mounted as ext4, read/write;
- `spi0.0` bound automatically to `apple-z2`;
- `/dev/input/event0` appeared automatically;
- the runtime touch IRQ was active; and
- libinput reported a 162 x 216 mm touch device.

The functional target test then performed four numbered taps, one one-finger
swipe, and two simultaneous two-finger swipes. The analyzer reported:

```text
7 contact strokes
4/4 taps at the requested targets
1/1 one-finger swipe
2/2 simultaneous two-finger swipes
297 SYN_REPORT events
6 balanced BTN_TOUCH press/release pairs
RESULT=PASS
```

After the test, NVMe remained live, the root filesystem remained read/write,
the touch device stayed bound, and the kernel log contained no SPI timeout,
touch-parser failure, DART fault, NVMe I/O error, kernel oops, or panic.

## Source layout

The final support is organized as a reviewable kernel series:

1. T8010 touch-clock binding and clock driver
2. Apple SIO binding, core, and DMAengine driver
3. T8010 SPI binding and generic controller implementation
4. J172 touchscreen binding and `apple_z2` support
5. T8010/J172 Device Tree description
6. MAINTAINERS entries

The key source files are:

```text
Documentation/devicetree/bindings/clock/apple,t8010-touch-clock.yaml
Documentation/devicetree/bindings/dma/apple,sio.yaml
Documentation/devicetree/bindings/spi/apple,spi.yaml
Documentation/devicetree/bindings/input/touchscreen/apple,z2-multitouch.yaml
drivers/clk/clk-apple-t8010-touch.c
drivers/soc/apple/apple-sio-core.c
drivers/dma/apple-sio-dma.c
drivers/spi/spi-apple.c
drivers/input/touchscreen/apple_z2.c
arch/arm64/boot/dts/apple/t8010.dtsi
arch/arm64/boot/dts/apple/t8010-pmgr.dtsi
arch/arm64/boot/dts/apple/t8010-ipad7.dtsi
```

## Current limitations

- The implementation is hardware-tested on J172. Other iPad 7 variants need
  their own board validation and, where applicable, Device Tree data.
- Matching Apple SmartIO and multitouch firmware must be supplied separately.
- The current full boot setup depends on PongoOS and the patched m1n1 loader.
- Suspend/resume and complete system power-management behavior have not yet
  received the same validation as cold boot and normal runtime input.

Within those boundaries, touch input is fully functional during normal Linux
desktop use and initializes automatically on every tested fresh boot.
