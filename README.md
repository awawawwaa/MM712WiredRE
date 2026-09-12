# MM712WiredRE — Cooler Master MM712 (wired version) protocol reverse engineering

Reverse engineering of the USB HID protocol used by the **Cooler Master MM712
wired version** (with a 3395 sensor, never released in most countries), as
driven by Cooler Master's MasterPlus+ software.

The main goal here isn't just to document the protocol — it's to replace
MasterPlus+ outright. Cooler Master's own software is bloated for what it
does: around 500 MB of RAM just sitting in the tray to let you change DPI.
This project's actual deliverable is a small, static webapp (`webapp/`) that
talks to the mouse directly over WebHID — no vendor software, no native
driver, and it works the same on Windows, Linux or macOS.

Inspired by [lx862's mouse reverse engineering
writeup](https://blog.lx862.com/blog/2024-05-13-reverse-engineering-a-mouse/)
and the [hyperx_pulsefire_dart_reverse_engineering
project](https://github.com/santeri3700/hyperx_pulsefire_dart_reverse_engineering).
Sister project: `../DeluxRE` (Delux M800 Mini Pro).

## Status

Everything this project set out to decode is confirmed, on real hardware, over
WebHID, with no native bridge:

* **DPI** — full table, per-stage enable, active-stage switching, both the full
  write and the 5-packet minimal switch. Formula confirmed end-to-end, 50 to
  26000 DPI.
* **Polling rate** — all 4 rates (1000/500/250/125 Hz), full write confirmed
  live.
* **LOD** — both settings, confirmed.
* **RGB** — Breathing, Static and Off, plus speed (Breathing), brightness and
  independent R/G/B, confirmed.
* **Debounce / Button Response Time** — deliberately retested with 5 explicit
  single-step changes; confirmed **absent** from this mouse, and confirmed to
  not survive a mouse power cycle even in Cooler Master's own software. It
  does nothing, so it isn't implemented here. See
  [Debounce time](#debounce-time--not-implemented-because-it-does-nothing).

`webapp/index.html` implements DPI, polling rate, LOD and RGB (Breathing/
Static/Off) with read + write, and every one of those has been run live
against the hardware and confirmed working — see
[Reference implementation](#reference-implementation).

Open ends, none blocking: the minimal single-packet write for polling rate
(shape known, untested), byte 6 of the `0x2b` RGB record, and Static's own
true default color (the one capture taken shows a value most likely carried
over from Breathing, not Static's factory default).

## Device identification

| | |
|---|---|
| Vendor ID | `0x2516` (Cooler Master) |
| Product ID | `0x01DF` |
| Connection | USB cable (wired mode). All findings below were captured over the cable. |
| Config | 1 configuration, 3 interfaces, bus-powered, 100 mA |

## USB/HID topology

Full configuration descriptor recovered from the capture
(`09 02 5b 00 03 01 00 a0 32 ...`):

| Interface | Class | Sub/Proto | Report descriptor | Endpoints | Role |
|---|---|---|---|---|---|
| 0 | HID (3) | Boot(1) / Mouse(2) | 83 bytes | `0x81` IN, int, 16 B, 1 ms | The actual mouse. Claimed exclusively by the OS mouse driver. |
| **1** | **HID (3)** | **0 / 0** | **34 bytes** | **`0x83` IN, int, 64 B, 1 ms**<br>**`0x04` OUT, int, 64 B, 1 ms** | **All vendor config traffic. This is the one you want.** |
| 2 | HID (3) | 0 / 0 | 134 bytes | `0x82` IN, int, 64 B, 1 ms | Not exercised in these captures (likely consumer/multimedia keys). |

### Confirmed: no native bridge needed

On the Delux M800 Mini Pro, the config reports lived behind a Generic Desktop
collection, which Chromium hard-blocks — forcing a native bridge process. Here,
interface 1 is HID **subclass 0, protocol 0**, with a **34-byte** report
descriptor — confirmed against real hardware to be exactly what that size
implied: a single **vendor-defined top-level collection**
(usage page `0xFF00`, usage `0x01`), one 64-byte input report and one 64-byte
output report, both **report ID 0**.

`navigator.hid` reaches it directly. **No native bridge, no driver — a real
static webapp**, confirmed with `sendReport(0, new Uint8Array(64))`.

One quirk worth knowing when writing the pairing code: Chrome/Windows present
this composite device as **three separate `HIDDevice` objects** in the picker
— one per USB interface, matching the topology table above. Interfaces 0 and 2
(the actual mouse, and the consumer-control collection) show up with Generic
Desktop / Consumer usage pages and get filtered out automatically; the one to
open is the one whose sole collection has usage page `0xFF00`. Its `productName`
is not "MM712" but the underlying **Cypress** USB controller's serial-style
string (e.g. `CY04055801A00004`) — a useful hint that this is a Cypress-based
HID controller, worth keeping in mind if register-level docs are ever needed.
`webapp/index.html` already picks the right one by scanning for the 64-byte
in/out pair rather than by name.

## Transport

Everything is a **64-byte interrupt transfer**:

* Host to device on endpoint `0x04`
* Device to host on endpoint `0x83`

Properties that make this pleasant to reimplement:

* **Fixed 64-byte packets**, zero-padded. Never shorter, never longer.
* **No report ID.** The payload starts at byte 0 with the opcode. In WebHID
  terms this is report ID `0`, i.e. `device.sendReport(0, new Uint8Array(64))`.
* **No checksum.** Not anywhere in the 64 bytes. (The Delux needed one; this
  does not.)
* **Every request is echoed.** The device replies on `0x83` with the same
  packet, with a few fields filled in by the device. That echo is the ACK —
  wait for it before sending the next packet.

> **Live-traffic finding (confirmed on hardware, not in the static captures):**
> the device also pushes **unsolicited** reports on `0x83`, opcode `0x43`
> (`43 01 00 00 00 c0` / `43 01 00 00 00 40`, alternating roughly every
> ~100 ms; occasionally `43 01 00 00 01 c0/40` — meaning not decoded), with no
> corresponding write. This looks like a periodic status heartbeat. It shares
> the same IN endpoint as command ACKs, so **code that treats every inbound
> report as the reply to the last request will get out of sync by one packet
> as soon as a heartbeat lands mid-transaction** — `webapp/index.html` matches
> replies against the opcode it just sent (`byte[0]` of the reply must equal
> `byte[0]` of the request, and must not be `0x43`) specifically to avoid this.
> The mis-synced version still worked in practice (USB preserves OUT packet
> order regardless of when the host reacts to IN traffic), but do not rely on
> that — match by opcode. **The fix is in place in `webapp/index.html` and
> verified live:** a full apply transaction now shows a clean 1:1 `>>`/`<<`
> pairing with every heartbeat correctly sidelined.
>
> The heartbeat's last byte flips `0x00`/`0x01` more often during physical
> mouse activity in testing (clicks/movement while the transaction ran) than
> at idle — consistent with it carrying some kind of live input or button
> state, though this is not decoded.

## Command framing

```
byte 0   : opcode / class
byte 1   : sub-command
byte 2.. : arguments / payload
```

The important pair:

| byte 0 | Meaning |
|---|---|
| `0x51` | **SET** — write config |
| `0x52` | **GET** — read config |

`0x51` and `0x52` share the *same* sub-command space, so every setting below is
both readable and writable by flipping byte 0. This is what makes a
read-modify-write webapp straightforward.

Other opcodes seen (session/framing, mostly only in the MasterPlus startup
capture):

| Packet | Observations |
|---|---|
| `41 80` | Sent before essentially every transaction. Echoed. Ping / "begin transaction". |
| `41 01`, `41 03`, `41 00` | Session control; `41 01` then `50 55` then `41` closes every apply. |
| `42 ...` | App-startup handshake. Only seen on first connect in the capture. |
| `43 01 00 00 00 <c0\|40>` | **Not handshake — a continuous unsolicited heartbeat**, confirmed on live hardware, pushed on the IN endpoint roughly every ~100 ms regardless of host activity. See the live-traffic note above. |
| `50 55` | Last write of every apply cycle. Almost certainly **commit/save to flash**. |
| `40 61` | Returns 44 bytes of `0xff`. Polled repeatedly. Unknown — likely macro or button-mapping memory, empty here. |

Known sub-commands (byte 1):

| Sub-cmd | Meaning | Confidence |
|---|---|---|
| `0x41` | DPI stage table + active stage | **confirmed** |
| `0xf0` | Polling rate | **confirmed** |
| `0x28` | Active RGB effect | **confirmed** |
| `0x2b` | Per-effect RGB parameters (speed, brightness, R/G/B) | **confirmed** — one byte (6) still unknown |
| `0x10` | Device status/info, read-only, 16-byte reply | unknown contents |
| `0x20` | Indexed register read (`52 20 <addr>`), addrs `b0`-`b5`, `be`, `bf`, `d8` | unknown |
| `0x22` | `52 22` returns `c0 c0` | unknown |
| `0x9c` | 23-byte config block, byte-identical in all 8 captures | unknown |
| `0xa8` | `52 a8` returns empty | unknown |

## DPI — sub-command `0x41`

### Stage record

Read: `52 41 <stage>` returns a 64-byte reply.
Write: `51 41 <stage> 00 00 <payload>`.

```
  0    1    2    3     4     5    6    7    8    9   10   11   12   13   14 15   16 17   18..63
 51   41  stg   00  actv    07   0a   0c   2d   00   f8   0f   89   bc   dpiX     dpiY    00...
```

| Offset | Field | Encoding | Confidence |
|---|---|---|---|
| 0 | opcode | `0x51` write / `0x52` read | confirmed |
| 1 | sub-command | `0x41` | confirmed |
| 2 | stage index | `0x00`-`0x06` (7 slots) | confirmed |
| 3 | record selector | `0x00` = stage record; `0xff` = active-stage record (see below) | confirmed |
| 4 | active stage | Host sends `0x00` on write; **device fills in the currently active stage** on every reply | confirmed, 8 captures |
| 5 | ? | always `0x07` | never varied |
| 6 | ? | always `0x0a` | never varied |
| 7 | ? | `0x10` factory, `0x0c` after MasterPlus' first apply, then constant | never varied after |
| 8 | ? | `0xff` factory, `0x2d` after MasterPlus' first apply, then constant | never varied after |
| 9 | ? | always `0x00` | never varied |
| 10 | **stage enabled** | bit `0x08`: `0xf8` = stage enabled, `0xf0` = stage disabled | **confirmed** |
| 11 | **LOD** | bit `0x40`: `0x0f` = low, `0x4f` = high | **confirmed** |
| 12 | ? | always `0x89` | never varied |
| 13 | ? | `0x8c` factory, `0xbc` after MasterPlus' first apply, then constant | never varied after |
| 14-15 | **DPI X** | little-endian 16-bit, `dpi = (n + 1) * 50` | **confirmed** |
| 16-17 | **DPI Y** | little-endian 16-bit, same encoding; always equal to X in every capture | confirmed |
| 18-63 | padding | `0x00` | confirmed |

### The DPI table on this device

| Stage | Raw | DPI |
|---|---|---|
| 0 | `07 00` | 400 |
| 1 | `0f 00` | 800 |
| 2 | `17 00` | 1200 |
| 3 | `1f 00` | 1600 |
| 4 | `3f 00` | 3200 |
| 5 | `9f 00` | 8000 |
| 6 | `07 02` | 26000 |

400 / 800 / 1200 / 1600 / 3200 / 8000 / 26000 is exactly the stock preset shown
in MasterPlus, and 26000 is this mouse's sensor maximum — confirmed against the
hardware by the device owner. That is a strong independent check: the formula
holds across the entire range, from the 400 DPI floor to the 26000 DPI ceiling,
and it confirms the DPI field really is **16-bit little-endian** (26000 needs
`n = 519`, which does not fit in one byte).

**How the formula and the 0-based indexing were pinned down** — three
independent captures, each of which only works under `dpi = (n+1)*50` with
0-based stage indices:

* `desselecting-the-800dpi-step...` — cycle 1 disables stage **1** (`0xf8` to
  `0xf0` on the record whose raw DPI is `0x0f`). The filename says the *800 DPI*
  step was deselected, so stage 1 = 800 DPI, so `0x0f` (15) maps to 800, i.e.
  `(15+1)*50`.
* Same capture, cycle 3 — active stage goes `02` to `01`, raw `0x17` to `0x0f`,
  i.e. 1200 to 800, exactly as the filename says.
* `from-800dpi-to-1600dpi` — active stage goes `01` to `03`, raw `0x0f` to
  `0x1f`, i.e. 800 to 1600 under the same formula.

The competing hypothesis `(n+1)*100` fails all three.

Encoding both ways:

```
raw = dpi / 50 - 1        dpi = (raw + 1) * 50
```

DPI is settable in steps of 50, from 50 up to 26000 (`raw` 0 to 519).

### Active stage record

```
51 41 00 ff <active_stage> 07      (write)
52 41 00 ff                        (read) -> 52 41 00 ff <active_stage> 07
```

* `<active_stage>` — 0-based index into the table above. **This is the packet a
  DPI-switching webapp actually needs.**
* trailing `0x07` — constant in every capture; most likely the number of stage
  slots (7), matching the 7 stage records.

Observed values: `01`, `02`, `03` — always consistent with the filename of the
capture and with the active stage the device reported back in byte 4 of the
preceding reads.

## Polling rate — sub-command `0xf0`

```
51 f0 00 00 <divisor>
52 f0                 -> 52 f0 00 00 <divisor>
```

| Divisor | Rate | Source |
|---|---|---|
| `0x01` | 1000 Hz | confirmed (6 captures) |
| `0x02` | 500 Hz | confirmed (`pollingrate-from-1000hz-to-500hz`) |
| `0x04` | 250 Hz | confirmed (`pollingrate-from-500hz-to-250hz`) |
| `0x08` | 125 Hz | **confirmed** — written live via `webapp/index.html` ("Write all settings"), device behaved correctly at 125 Hz |

i.e. `rate = 1000 / divisor`.

## Lift-off distance

LOD is **not** its own command — it is bit `0x40` of byte 11 of *every* DPI
stage record, and MasterPlus writes the same value into all 7 records.

| Byte 11 | LOD |
|---|---|
| `0x0f` | Low |
| `0x4f` | High |

Confirmed by `lod-from-low-to-high` (writes `0x4f`) and `lod-from-high-to-low`
(writes `0x0f`), which are otherwise byte-identical captures.

## RGB

Two sub-commands. **Correction:** an earlier version of this document had the
`0x2b` byte layout wrong — it read bytes 7-9 as RGB and called bytes 10-12 a
constant trailer. Five follow-up captures that each varied exactly one
parameter (speed, brightness, R, G, B in turn) prove the opposite: bytes 7-8
are the constant, byte 9 is brightness, and RGB is at **bytes 10-12**. The
table below is the corrected layout.

This project covers **Breathing**, **Static** and **Off**.

**`0x28` — select the active effect**

```
51 28 00 00 <effect_id>
```

| `effect_id` | Effect | Confidence |
|---|---|---|
| `0x00` | Static | confirmed — `rgb-from-breathing-to-static` |
| `0x01` | Breathing | confirmed |
| `0xfe` | Off / none | confirmed |

`0x00` was the "unnamed effect only seen at startup enumeration" in an earlier
version of this document — `rgb-from-breathing-to-static` names it: it's
**Static**, matching Cooler Master's own software, where Static only exposes
color and brightness (no speed control at all, unlike Breathing).

**`0x2b` — per-effect parameters**

```
  0    1    2    3     4      5      6     7    8     9     10   11   12
 51   2b   00   00   eff   speed   ??   ff   ff   bri    R    G    B
```

| Offset | Field | Encoding | Confidence |
|---|---|---|---|
| 4 | effect id | as above | confirmed |
| 5 | **speed** | lower = faster; Breathing only, see table below | **confirmed for Breathing** — no UI control maps to it for Static |
| 6 | unknown | `0x20` seen for Breathing, `0x00` for Static | unconfirmed |
| 7 | unknown | always `0xff` in every capture | never varied |
| 8 | unknown | always `0xff` in every capture | never varied |
| 9 | **brightness** | `0x00`-`0xff`, `0xff` = 100% | **confirmed** |
| 10 | **R** | `0x00`-`0xff` | **confirmed** |
| 11 | **G** | `0x00`-`0xff` | **confirmed** |
| 12 | **B** | `0x00`-`0xff` | **confirmed** |

### Speed — confirmed, all 5 levels (Breathing only)

`rgb-speed-from3of5-to-2of5-to-1of5-back-to-3` and
`rgb-speed-from3of5-to-4of5-to-5of5-back-to-3` between them exercise every
slider position on the Breathing effect:

| Speed (of 5) | Byte 5 | Decimal |
|---|---|---|
| 1 (slowest) | `0x3c` | 60 |
| 2 | `0x37` | 55 |
| 3 (default) | `0x31` | 49 |
| 4 | `0x2c` | 44 |
| 5 (fastest) | `0x26` | 38 |

No clean formula fits (steps alternate -5/-6), so this is given as a lookup
table, not a computed value. One of the two captures shows an extra
intermediate apply at level 2 while dragging back from 1 to 3 — consistent
with MasterPlus applying on every step of a slider drag, not just on release.
Static has no speed control in Cooler Master's own software, so this table
doesn't apply to it.

### Brightness — confirmed

`rgb-brightness-from-full-to-kindahalf-to-kindalow-tohighagain` varies byte 9
only (on Breathing), R/G/B held fixed at `ff 32 3c`:

| Step | Byte 9 |
|---|---|
| Full | `0xff` |
| "kinda half" | `0x80` (128, ~50%) |
| "kinda low" | `0x13` (19, ~7%) |
| High again | `0xff` |

### Color — confirmed, all three channels independently

`rgb-color-rfrom240to255-then-gfrom0to50-then-bfrom255to60` changes one
channel at a time (on Breathing), starting from the device's existing color
(`R=0xf0 G=0x00 B=0xff`, the same default seen in every earlier capture — that
default was never actually a "constant trailer", it is simply the RGB value
nobody had changed yet):

| Step | Byte 10 (R) | Byte 11 (G) | Byte 12 (B) |
|---|---|---|---|
| start | `f0` (240) | `00` (0) | `ff` (255) |
| R → 255 | `ff` (255) | `00` | `ff` |
| G → 50 | `ff` | `32` (50) | `ff` |
| B → 60 | `ff` | `32` | `3c` (60) |

Each step is an independent apply cycle that rewrites only the channel that
changed — R, then G, then B — exactly matching the filename.

### Static (`0x00`) — confirmed, with one open discrepancy

`rgb-from-breathing-to-static.pcapng` (Breathing set to blue, switching to
Static) wrote:

```
51 2b 00 00 00 03 00 ff ff ff 00 00 ff
```

i.e. effect=`00`, byte5=`03`, byte6=`00`, brightness=`ff` (full), **R=`00` G=`00`
B=`ff` — blue**, not the green the person making the capture says Static had
pre-set in Cooler Master's software before this change.

This is most likely the same behavior already noted for the other effects:
MasterPlus appears to carry over one shared "currently edited" color across
effect tabs rather than always writing each effect's own distinct stored
value — the blue written here matches Breathing's color at the time (per the
person testing), not Static's own supposed green. Byte5 (`0x03`) has no UI
control in Static's panel (no speed slider) and is likely just an unused
default, not a real parameter for this effect. Take Static's own "true"
default color/byte5 as **not yet confirmed** — only the byte *positions* are
confirmed (same layout as Breathing), not what a fresh, never-touched Static
record contains.

The effect-select write for this capture also confirms `0x28` accepts
implicit zero: it was sent as bare `51 28` (2 bytes) rather than
`51 28 00 00 00` — the trailing zero bytes (including the `0x00` effect id
itself) are simply omitted since they're already zero-padding. Functionally
identical to every other `51 28 00 00 <id>` write.

Turning Breathing or Static on writes the `0x2b` record first, then the
`0x28` selection. Turning any effect off writes only `51 28 00 00 fe`, no
`0x2b`.

## Debounce time — not implemented, because it does nothing

Not implemented in `webapp/index.html` on purpose. Beyond it being absent
from the wire (below), setting Button Response Time/debounce in Cooler
Master's own software doesn't survive a power cycle either — every value
tried was back to the 6 ms default after restarting the mouse. The control
does nothing, on the device's own vendor software, not just over this
protocol.

### Also confirmed absent from this interface

`debouncetime-from-6ms-to-1ms.pcapng` contains five apply cycles, one per
step from 6 ms down to 1 ms — all five byte-for-byte identical across all 170
packets.

`pollingrate-decrease-by-1ms.pcapng` (misleading filename; it's actually the
same test, decremented one step at a time) confirms it again: 5 separate,
deliberately-triggered 17-packet apply transactions, still byte-for-byte
identical to each other.

Two of the three sub-commands MasterPlus writes on every apply (`0x41` and
`0xf0`) are already fully mapped, and the third (`0x9c`) is a constant across
every capture taken, factory value included. There is no fourth write —
debounce is simply not sent to the mouse over this interface.

## The apply transaction

Every settings change in MasterPlus is the same fixed **17-packet** transaction
(each packet echoed by the device before the next is sent). Verbatim, from
`lod-from-low-to-high`, trailing zero padding to 64 bytes omitted:

```
 1  41 80                                                ping
 2  52                                                   (bare read, empty reply)
 3  41 80                                                ping
 4  51 41 00 00 00 07 0a 0c 2d 00 f8 4f 89 bc 07 00 07   stage 0
 5  51 41 01 00 00 07 0a 0c 2d 00 f8 4f 89 bc 0f 00 0f   stage 1
 6  51 41 02 00 00 07 0a 0c 2d 00 f8 4f 89 bc 17 00 17   stage 2
 7  51 41 03 00 00 07 0a 0c 2d 00 f8 4f 89 bc 1f 00 1f   stage 3
 8  51 41 04 00 00 07 0a 0c 2d 00 f8 4f 89 bc 3f 00 3f   stage 4
 9  51 41 05 00 00 07 0a 0c 2d 00 f8 4f 89 bc 9f 00 9f   stage 5
10  51 41 06 00 00 07 0a 0c 2d 00 f8 4f 89 bc 07 02 07 02   stage 6
11  51 9c 00 00 e7 cc 00 00 00 ff f0 00 ff 00 ff 00 00 e0 ff ff 75 00 ff   (unknown block)
12  51 41 00 ff 02 07                                    active stage = 2
13  51 f0 00 00 01                                       polling = 1000 Hz
14  52 10                                                read status
15  41 01
16  50 55                                                commit / save
17  41
```

MasterPlus always rewrites the **entire** configuration, even when one checkbox
changed. That is its habit, not necessarily a requirement of the device.

### Minimal DPI change — confirmed working

```
41 80                    ping
51 41 00 ff <stage> 07   set active stage
41 01
50 55                    commit
41
```

**Confirmed on real hardware** via `webapp/index.html`'s "Switch active stage
only" button — a single 5-packet transaction reliably switches which DPI stage
is active, with no need to resend the DPI table.

This packet **only** changes which stage is active. It does not touch the
per-stage DPI values (`51 41 <stage> 00 00 ...`) or the polling rate (`51 f0`)
— those need their own write if you want to change them, and, per the pattern
above, are almost certainly just as cheap to write on their own:

```
41 80                 ping
51 f0 00 00 <div>     set polling rate
41 01
50 55                 commit
41
```

This polling-only form has **not been tried yet** — `webapp/index.html`
currently only offers the full 17-packet write for polling rate (see
"Write all settings"), confirmed working. If a minimal polling switch is
wanted later, this is the shape to try.

## Recommended next steps

1. ~~Confirm WebHID reach.~~ **Done** — see [Confirmed: no native bridge
   needed](#confirmed-no-native-bridge-needed).
2. ~~Replay the full apply transaction live.~~ **Done** — DPI and polling rate
   both changed correctly on real hardware through `webapp/index.html`.
3. ~~Try the minimal single-packet DPI switch.~~ **Done** — confirmed working,
   see [Minimal DPI change](#minimal-dpi-change--confirmed-working).
4. **Try the equivalent minimal write for polling rate** (shape given in the
   same section) — not yet attempted.
5. ~~Re-capture debounce.~~ **Done** — confirmed it does nothing at all, see
   [Debounce time](#debounce-time--not-implemented-because-it-does-nothing).
6. ~~Capture 125 Hz polling to confirm `0x08`.~~ **Done** — confirmed working
   live.
7. ~~Vary RGB speed, brightness and color; name every effect.~~ **Done** — see
   [RGB](#rgb). Static's true default color is still open.

## Reference implementation

* `webapp/index.html` — the control tool. Reads the live configuration, lets
  you edit the 7 DPI stages / active stage / polling rate / LOD and write it
  back, plus a separate RGB panel (effect, speed, brightness, color) that
  applies through its own capture-matched transaction. Every packet it sends
  and receives is shown in a hex log so it can be diffed against the raw USB
  captures used to reverse-engineer the protocol.

It's a single self-contained file with no build step and no dependencies.
