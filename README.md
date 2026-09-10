# AgileX SCOUT MINI, from the outside

Notes and tools from getting an **AgileX SCOUT MINI OMNI** to talk without the
vendor SDK — over CAN and over the RS232 port next to it. Written while debugging a
unit that appeared dead and turned out not to be.

Everything here is what was measured on one vehicle. Where something is a guess it
says so, and where a conclusion was reached and later overturned, the overturning is
recorded rather than the file being quietly edited.

## Where this got to

| link | telemetry | control |
|---|---|---|
| **CAN** | never received a single frame on this unit | not reached |
| **RS232** | works, decoded, running continuously | **does not work** — mode and drive commands have no effect |

The vehicle drives fine from its own remote throughout. So the fault is not the
chassis, and the remote working is not evidence that serial control is supported —
those are different paths, and conflating them cost time here.

## The four things worth taking

**1. The RS232 frame format, and its checksum.**

```
5A A5 | len | type | id | data | frame id | checksum
```

`len` counts from itself through the frame id, `type` is `0x55` control / `0xAA`
feedback, and `checksum` is the sum of every preceding byte masked to 8 bits. The
checksum rule was found by brute force over a capture before the manual turned up —
6637 of 6637 frames agreed — and the manual then said the same thing. Worth knowing
that the near-miss rules scored 30 and 37 out of 6637: a wrong rule still hits
sometimes, so "it matches a few" means nothing.

**2. A lone CANable 2.0 can be self-tested, with nothing else on the bus.**

CAN needs a second node to acknowledge a frame, so the obvious test cannot be run by
one adapter, and no arrangement of CAN_H / CAN_L / GND changes that. The adapter will
still tell you, through the error register its firmware exposes as `E` (not the
standard slcan `F`, which this firmware does not implement):

```
0x00  fresh power-on
0x10  after 200 frames at 1 kHz  ->  FULLBUF_CANTX
```

Bit 4 setting is the **pass** condition, which reads backwards until you follow the
firmware. `can_process()` offers a frame to the peripheral only while
`HAL_FDCAN_GetTxFifoFreeLevel() > 0`, and advances its own tail either way. A dead
peripheral leaves the hardware FIFO reading empty forever, so every frame is offered,
refused, and dropped — the software queue drains and bit 4 never sets. The queue can
only overflow if the hardware FIFO is full and *staying* full, and the only thing
that keeps it full is a peripheral that accepted frames and is retransmitting them
because nothing acknowledges.

So: the controller is initialised, it took the frames, it is retrying into an empty
bus. Everything up to the transceiver works. `tools/selftest.py` runs this.

**The same run tests the vehicle.** Attached to a chassis that is powered and on the
bus, its controller acknowledges, the FIFO drains, and bit 4 stays clear. Bit 4
setting *with the vehicle attached* means the vehicle is not acknowledging — which
separates "our side is dead" from "the vehicle is not on the bus" without receiving
a single frame from it.

**3. The vehicle was not broken.**

The working conclusion after CAN stayed silent through every wiring configuration was
that the chassis had failed. RS232 disproved it: 25.0 V battery, four motors
reporting 29 °C, motion feedback the moment the sticks moved. Reaching for the
second interface was worth more than any further CAN rewiring.

**4. There is no documented serial control path.**

The vendor SDK contains zero occurrences of "serial", the `agilex_firmware`
repository is a 404, and on this unit byte 1 of the system feedback frame — the
"control mode" field per the SCOUT 2.0 manual — never changed value even while the
remote was plainly driving. That makes any safety gate built on reading the mode a
gate on a number that means nothing, which is why `agxserial.py` documents the gate
as unverifiable rather than pretending it works.

## Tools

All host-side Python, no install step, no router required.

| | |
|---|---|
| `tools/selftest.py` | is the CANable's CAN controller alive? Works with nothing attached. |
| `tools/canweb.py` | the CAN bus in a browser — frame table, IDs decoded from `doc/CAN.md` |
| `tools/serweb.py` | the RS232 telemetry in a browser — battery, motion, four actuators |
| `tools/agxserial.py` | bidirectional RS232: watch, enable, standby, drive, clear |
| `tools/canwatch.py` `canshow.py` `canscan2.py` `canauto.py` `hunt.py` | bring-up scratch tools, kept because they record what was tried |

`agxserial.py drive` refuses to start unless the chassis is already in serial mode,
sends continuously while driving, and sends zeros on the way out rather than relying
on the chassis' 500 ms timeout. The timeout is the backstop, not the plan.

## Router-side packages

`can-bridge/` and `agx-cmd/` are OpenWrt packages — they put the vehicle's CAN
traffic on the network and take commands back. They are here rather than in the
router repository because the vehicle outlives the router in this setup. Build them
by adding this repository as a feed; they have no dependency on any particular
router beyond SocketCAN.

The router they were written for is
[a3004-sensor-bridge](https://github.com/hwkim3330/a3004-sensor-bridge).

## What is still not known

- Whether the CAN wiring to the vehicle is correct. The connections were temporary
  contacts, never soldered, and that is the one untested segment: the adapter is
  proven good and the chassis is proven alive, so the wire between them is what is
  left. `selftest.py` settles it once the joints are made.
- Whether the vehicle transmits on CAN at all.
- The OMNI's lateral velocity. The SCOUT 2.0 control frame marks bytes 4 and 5
  "reserved" and the OMNI has an axis the 2.0 does not, so those bytes are plausibly
  it. That is a guess, so `agxserial.py` sends them as zero.

## Reading

- [`doc/CAN.md`](doc/CAN.md) — the CAN protocol, the IDs, and the full record of what
  was tried and ruled out
- [`doc/SCOUT-FIRMWARE.md`](doc/SCOUT-FIRMWARE.md) — firmware and serial-number
  findings

## Licence

GPL-2.0-or-later, matching the SPDX headers on the C sources.
