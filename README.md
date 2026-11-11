# mcp2518fd-nv

A driver for the MCP2518FD, an external CAN FD controller with an SPI
interface from Microchip, specified in the data sheet "MCP2518FD: External
CAN FD Controller with SPI Interface", document DS20006027B, December 2020,
and in the "MCP25XXFD Family Reference Manual", DS20005678E
([microchip.com/en-us/product/MCP2518FD](https://www.microchip.com/en-us/product/MCP2518FD)).
The driver implements the `CanBus` trait of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for classic
frames over an SPI bus, a chip-select pin and an optional interrupt pin, so a
board with no CAN controller of its own gets a CAN bus on any SPI port.  FD
frames, bit rate switching, the acceptance filters, the transmit FIFO, the
transmit event FIFO, the interrupt flags and the GPIO pins are the package's
own functions beside the trait.

**Hardware: NOT VERIFIED.**  No board carries an MCP2518FD yet.  The device
program below builds for the nRF52840 DK and the nRF54LM20 DK.  The driver's
functions run over a double of the part that follows the data sheet's
register map, its message object layout and its SPI instructions.  A frame
from a part on a board has not been sent or received.

## Install

```sh
novo pkg add mcp2518fd-nv
```

The device program is `examples/can_fd_echo.nv`.  Make it the `main` of a
package that depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/can_fd_echo.nv` is a node on a CAN FD bus behind an MCP2518FD
breakout.  It opens the part at 500 kbit/s with a 2 Mbit/s data phase, sends
an FD frame with the identifier 0x123, bit rate switching and 12 data bytes
once a second, and prints every frame it receives over the board's UART:

```
rx id=291 len=12 fd=1
```

```novo norun:needs-pkg
use embedded_hal.{ BoardSpi, BoardGpio }
use dev_bus
use canframe
use mcp2518fd
use mcp2518fd.{ Mcp2518 }

@tier(embedded)
fn main() [hw]
    // Chip select on pin 28, INT on pin 29, a 40 MHz crystal.
    let w = dev_bus.wiring_irq(dev_bus.wiring_spi(0, 28), 29)
    let can = Mcp2518.attach(BoardSpi { id: 0 }, BoardGpio { id: 0 }, w, 40000000)
    hal.timer.delay_ms(10)
    if can.open_with(500000, 2000000, mcp2518fd.config_default()) != 0
        hal.uart.write("mcp2518fd: no answer\n")
        return
    loop
        var inb: Vec[u8; 64] = Vec.new()
        let h = can.receive_fd(var inb)
        if canframe.fd_is_frame(h)
            hal.uart.write(str.from_int(h.id))
            hal.uart.write("\n")
        hal.timer.delay_ms(100)
```

The full program, which also sends the counter and reads the transmit events
the part keeps, is `examples/can_fd_echo.nv`.
A script builds it for the boards below by making it the `main` of a scratch
package that depends on this one.  It also runs the emulated board's image
under QEMU, where no part answers, and checks that the program reports it.

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which the program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 19,540 B | 11,690 B | 3,196 B |
| `nrf54lm20-dk` | 20,988 B | 12,322 B | 3,172 B |

The numbers are from the build's memory report.  Flash also holds the
runtime, the compiler's support library and the program's tables.  The arena,
65,536 B on the Nordic boards, is the runtime's heap and is reserved whether or
not the program uses it.  The Nordic DKs drive SPI with a SPIM (the nRF52840
DK's SPIM2 on SCK P1.15, MISO P1.14 and MOSI P1.13, the nRF54LM20 DK's SPIM21
on SCK P1.07, MOSI P1.06 and MISO P1.05), on the pins `docs/hal/README.md`
names under "The Nordic DKs' buses", and the chip select is a GPIO pin the
caller drives, so both rows are the image with the driver's whole path in it.
With no part on the bus the program prints `mcp2518fd: no answer`.  A reply
from the part over a DK's bus is NOT VERIFIED until the driver rig carries
one.

The driver's own code builds no list: every buffer is a fixed-size `Vec` in
the caller's frame, the largest 96 bytes, and a received FD frame's payload is
a `Vec[u8; 64]` the caller owns.

### The effects

Every function of the driver that reaches the bus, the pins or the timer
declares `[hw]`, and nothing else.  The bit timing, the object words and the
SPI clock limit are pure.  The driver performs no input or output of its own
and allocates nothing of its own.  The package checks at the embedded tier,
so a program of any tier may call it.  The program above declares `[hw]` for
the same reason.

## What it is

The MCP2518FD implements CAN FD (ISO 11898-1:2015) and classic CAN 2.0B:
standard (11-bit) and extended (29-bit) identifiers, data frames of up to 64
bytes, remote frames in classic format, and a data phase at up to 8 Mbit/s
after a nominal phase of up to 1 Mbit/s.  It speaks SPI in mode 0,0 or 1,1.
The driver uses mode 0.

The part's instructions are a command in the high nibble of the first byte and
a 12-bit address in the rest of two bytes.

| Instruction | Command | What it does |
|---|---|---|
| RESET | 0x0, address 0x000 | the registers to their reset values, Configuration mode |
| READ | 0x3 | bytes from an address, the address advancing |
| WRITE | 0x2 | bytes to an address, the address advancing |
| READ_CRC, WRITE_CRC, WRITE_SAFE | 0xB, 0xA, 0xC | the same with a CRC-16 on the SPI frame. NOT IMPLEMENTED |

Registers are 32 bits, the low byte at the low address.  A register of the CAN
module is at 0x000 to 0x2EF, the 2 KB message RAM at 0x400 to 0xBFF and the
part's own registers (OSC, IOCON, ECCCON) at 0xE00.  A RAM access is in whole
words at a word-aligned address.

The driver configures one layout of the message RAM and does not offer
another: a transmit event FIFO of 8 objects of 8 bytes, a transmit queue of 4
objects, FIFO 1 as the receive FIFO of 16 objects and FIFO 2 as a transmit
FIFO of 4 objects, each of the last three with a 64-byte payload and so 72
bytes an object.  That is 64 + 288 + 1152 + 288 = 1792 of the 2048 bytes.

| Words of an object | Holds |
|---|---|
| T0, R0 | SID in bits 10:0, EID in bits 28:11 |
| T1 | DLC, IDE, RTR, BRS, FDF, ESI in bits 8:0, the sequence number in bits 31:9 |
| R1 | DLC, IDE, RTR, BRS, FDF, ESI in bits 8:0, the filter that hit in bits 15:11 |
| T2.., R2.. | the data bytes, byte 0 in the low byte of the first word |

An extended identifier is split: its top 11 bits are the SID and its low 18
bits the EID.  The identifier 0x18DAF110 has the SID 0x636 and the EID
0x2F110, so T0 is 0x2F110 << 11 | 0x636 = 0x17888636.  The driver does not
compute an object's address: it reads it from the user address register of the
FIFO, which holds an offset from 0x400.

The bit rate is SYSCLK / (BRP x (1 + TSEG1 + TSEG2)) in each phase, with the
registers holding every length less one.  The solver picks the smallest
prescaler that makes the rate exactly with the bit in the range the reference
manual gives, and puts the sample point at 80 % of the bit.  The registers it
finds for the two crystals the part takes at the common rates are in the table.

| System clock | Nominal 500 kbit/s | Nominal 1 Mbit/s | Data 2 Mbit/s | Data 5 Mbit/s |
|---|---|---|---|---|
| 40 MHz | CiNBTCFG 0x003E0F0F | 0x001E0707 | CiDBTCFG 0x000E0303 | 0x00040101 |
| 20 MHz | 0x001E0707 | 0x000E0303 | 0x00060101 | 0x00010000 |

The 40 MHz, 500 kbit/s and 2 Mbit/s values are the reference manual's own
example (its Table 3-5), and `timing_known` holds the table.  A data rate of
1 Mbit/s or more turns on automatic transmitter delay compensation with the
offset the manual gives, BRP times TSEG1, held to 63.

## What the package contains

| Module | What is in it |
|---|---|
| `mcp2518fd` | The bit timing, the object words, the SPI instructions over the trait `FdLink`, the modes, the reset and the oscillator, the layout, the filters, the interrupts, the transmit and receive functions, the transmit event FIFO, the error counters, the types `Mcp2518`, `Mcp2518Link`, `Mcp2518Config` and `Mcp2518Timing`, and the `CanBus` implementation |
| `mcp2518fd_irq` | `Mcp2518Line`, the INT pin as devices-nv's `AsyncDeviceReady` |

## How to choose an entry point

A program on a board uses `Mcp2518`, as the board's device table builds it,
with the crystal of the breakout:

```novo norun:fragment
    devices
        can  Mcp2518  from "mcp2518fd-nv"  on spi0  cs can_cs  irq can_int
```

A program that holds its own bus, or a test with a double of the part,
implements `FdLink` over its bus and calls the module's functions, which take
the link: `mcp2518fd.open_code(link, osc_hz, bitrate, data_rate, config)`,
`mcp2518fd.send_fd(link, queue, header, var payload, seq)`.  The framing of the
bus is `spi_ready` and `spi_exchange`, generic over `SpiBus` and `GpioOut`.

| Method of `Mcp2518` | What it does |
|---|---|
| `attach(bus, pins, w, osc_hz)` | the part with the chip select `w.cs` and, if the wiring has one, the INT pin `w.irq`; nothing is sent |
| `open(bitrate, mode)` | `CanBus`: Normal CAN 2.0 mode for `CAN_MODE_NORMAL`, internal loopback for `CAN_MODE_LOOPBACK`, classic frames only |
| `open_with(bitrate, data_rate, config)` | opens with the settings of `Mcp2518Config` and answers a code, 0 for success |
| `send(frame)`, `receive()` | `CanBus`: a classic frame into the transmit queue; the next classic frame |
| `tx_done()`, `bitrate()`, `errors()` | `CanBus`: no frame waiting and the part open; the rate read back from CiNBTCFG and OSC; the counters and state |
| `send_in(queue, frame, seq)`, `send_fd_in(queue, header, var payload, seq)` | a classic or an FD frame into the queue (0) or the transmit FIFO (1) with a sequence number |
| `receive_fd(var payload)`, `receive_by_pin()` | the next frame of either format; the next classic frame, reading the INT pin first |
| `event()` | the oldest transmit event: identifier, flags, length and the sequence number |
| `mode()`, `set_mode(m)` | the mode in CiCON.OPMOD; a mode request, confirmed |
| `set_filter(...)`, `clear_filter(n)` | an acceptance filter, and its removal |
| `flags()`, `enable_flags(mask)`, `clear_flags(mask)`, `int_level()` | CiINT; its enables; its clearing; the INT pin |
| `abort_all()` | aborts every frame waiting to be sent |
| `gpio_dir(pin, output)`, `gpio_write(pin, level)`, `gpio_read(pin)` | the part's GPIO0 and GPIO1 |
| `ecc()`, `devid()` | the ECC status register; the device ID register |

`Mcp2518Config` carries the mode (`MCP_MODE_NORMAL_FD`, `MCP_MODE_LOOPBACK`,
`MCP_MODE_LISTEN`, `MCP_MODE_LOOPBACK_EXT`, `MCP_MODE_NORMAL_20`,
`MCP_MODE_RESTRICTED`), the PLL, the SPI clock, the retransmission setting, whether
filter 0 passes every frame, the flags that drive the INT pin, whether
GPIO0 and GPIO1 are the INT0 and INT1 outputs, and the two sample points.

The `CanBus` trait has two modes and a `CanBusFrame` of 8 bytes; FD frames,
Listen-Only, the filters, the transmit priorities and the interrupt flags have
no place in it, so they are the driver's own functions beside it.  An FD
frame's header is `can-nv`'s `CanFdFrame`, with the identifier, the flags
(`CAN_FLAG_EXTENDED`, `CAN_FLAG_RTR`, `CAN_FD_FLAG_FD`, `CAN_FD_FLAG_BRS`,
`CAN_FD_FLAG_ESI`) and the length, and its payload is in the caller's
`Vec[u8; 64]`.  A receive that finds nothing answers `fd_none()`, and one
that finds the part gone answers `fd_fault(CAN_FAULT_DISCONNECTED)`.

### The awaitable form

The part signals a received frame, a finished transmit and an error on its INT
pin, which is open-drain and active low and stays low while an enabled flag is
set.  `mcp2518fd_irq.Mcp2518Line` holds the board's `AsyncEdge`, its GPIO and the
pin, and implements `AsyncDeviceReady`: `ready_arm(f)` arms a falling edge and
the board's interrupt resolves the Future `f`; the task awaits it with
`core.rt.future_await` and then reads the flags.  Because the pin is a level, a
flag set before the arm leaves no edge to wait for, so `ready_arm` reads the pin
first and resolves the Future at once when it is already low.  A task clears
what it handled before it arms again.  An `async fn` is refused at the embedded
tier the package checks at, which is why the form is the awaitable one.

## The rules a user needs

1. The crystal's frequency is an argument of `attach`: the part does not know
   it, and the bit rate is computed from it.  A breakout has 20 MHz or 40 MHz.
   A wrong value makes a wrong bit rate and no error.
2. `open_code` answers a code: 0 for success, -1 when the part does not answer
   or the wiring has no chip select, -2 for a mode other than 0, 2, 3, 5, 6 and
   7 or a retry setting outside 0 to 2, -3 for a rate the oscillator cannot
   make, -4 when a bit timing did not read back as written, -5 when the mode
   did not take within 50 ms, -6 when the oscillator did not come up.  An SPI
   bus no part drives reads 0xFF or 0, never the reset value of CiCON, so a
   missing part is -1.
3. The SPI clock is at most 0.85 times half the system clock.  The bus is opened
   at 17/40 of the oscillator, 17 MHz for 40 MHz and 8.5 MHz for 20 MHz, and
   raised to `spi_hz` once the oscillator is ready, never above that limit or
   20 MHz.  With a 4 MHz crystal and the PLL the bus starts at 1.7 MHz.
4. RESET is only valid in Configuration mode, so the driver requests it, with
   ABAT, before the RESET instruction.  It then waits 5 ms and polls CiCON for up
   to 30 ms until it reads its reset value 0x04980760.  A read while the
   oscillator restarts answers zeros.
5. The layout, the bit timing and the filters are written in Configuration mode,
   which is where the part is after RESET.  The bit fields of IOCON are written
   one byte at a time, because a 32-bit write clears GPIO latches that were not
   named.
6. ECC is enabled and all of the message RAM is written with zeros before the
   mode is left, so that no word is read before it has its parity bits.
7. A mode request is confirmed in CiCON.OPMOD.  A change to Normal mode waits for
   11 recessive bits, which is 1.1 ms at 10 kbit/s; `mode_set` polls every
   100 microseconds for 50 ms and answers whether the mode came.
8. The trait's `open` runs Normal CAN 2.0 mode, in which the part answers an FD
   frame on the bus with an error frame, because the trait carries classic
   frames only.  A node on a bus that carries FD frames opens with
   `MCP_MODE_NORMAL_FD`, through `open_with`.
9. The transmit queue sends the lowest identifier first among its frames; the
   transmit FIFO sends in the order frames were put in.  The queue has the
   higher priority.  `send` queues in the queue.  Both answer full when their
   objects are used: the code -2, with no object written.
10. A sequence number given to a send comes back in the transmit event.  The
    events are kept because CiCON.STEF is set.  A program that never reads
    them leaves the FIFO full; the next event is discarded and TEFOVIF is set,
    and transmission goes on (Reference Manual, Figure 8-27).
11. A receive reads CiFIFOSTA1 and CiFIFOUA1 in one 8-byte read, the object at
    0x400 plus the user address, and increments the tail with UINC.  A frame of
    more than 8 data bytes needs a second read.
12. `receive` on the trait drops an FD frame, because a `CanBusFrame` holds
    8 bytes, and tries the next, up to the depth of the FIFO.  `receive_fd`
    returns both formats.
13. A frame the bus does not acknowledge is sent again until it is, unless
    `retries` is 1 (three attempts) or 2 (one).  `abort_all` sets ABAT and waits
    for the part to clear it.
14. A filter is disabled before its object, mask and FIFO pointer are written,
    and the object and mask are read back.  `all_types` clears MIDE so the
    filter passes standard and extended frames; otherwise the filter matches the
    type its EXIDE says.
15. The INT pin follows the flags of `irq_mask`.  The flags of the FIFOs
    (RXIF, TXIF, TEFIF) follow the FIFOs and clear when the FIFO is read or
    written; the others (mode change, wake, bus error, system error, invalid
    message, time base) are cleared by `clear_flags`, which writes the bits to
    clear as 0.
16. In Configuration mode CiTREC reads bus-off, because the module is not on the
    bus.  `errors` answers error-active with zero counters there.
17. The oscillator is 40, 20 or 4 MHz.  A 4 MHz crystal with the PLL runs the
    system clock at 40 MHz.
18. `devid` returns the ID and revision byte and tests nothing.  The data sheet
    names the register as the way to tell the MCP2517FD from the MCP2518FD and
    gives no values.

## What is not included

- The CRC forms of the SPI instructions (READ_CRC, WRITE_CRC, WRITE_SAFE) and the
  CRC register. NOT IMPLEMENTED.
- Sleep and low-power mode, and wake-up by bus activity. NOT IMPLEMENTED.
- Time stamping: the time base counter, CiTSCON and the time stamps of received
  messages and transmit events. NOT IMPLEMENTED.
- The bus diagnostic registers CiBDIAG0 and CiBDIAG1. NOT IMPLEMENTED.
- A message RAM layout other than the one above, FIFOs 3 to 31, and the
  transmit bandwidth sharing delay. NOT IMPLEMENTED.
- DeviceNet filtering of data bytes, the start-of-frame signal on CLKO, the
  transceiver standby pin, the system clock divider and the CLKO divider.
  NOT IMPLEMENTED.
- An `Identifiable` form: the data sheet gives no ID values.
- An async form on the trait: `CanBus` has no `async fn` trait, and an `async fn`
  is refused at the embedded tier; the awaitable form is the INT pin above.
- The MCP2517FD and the MCP251863, which are other parts, and the MCP2515, which
  `mcp2515-nv` drives.
- A transmit queue, a receive ring and ISO-TP above the frames: those are
  [can-nv](https://novo-lang.org/packages/can-nv)'s.

## Reference drivers read

Each of these was read for its fixes, and none was ported.  The fix is named at
its site in `src/mcp2518fd.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/can/can_mcp251xfd.c` | the SPI clock clamped to 17/40 of the oscillator until the oscillator is ready; the 5 ms after RESET and the poll of CiCON for Configuration mode, 30 ms; the PLL settle of 200 µs; the single TXQSTA and TXQUA read, the object write, and UINC with TXREQ in one byte; the receive FIFO's status and user address read together; the errata note on IOCON; the transmitter delay compensation off for classic frames |
| `mcp2517` crate 0.4.1 | the mode request written to byte 3 of CiCON with ABAT, and a 2 ms wait that is too short for a change to Normal mode on a slow bus; the filter disabled before its pointer is written; IOCON written a byte at a time; a receive object with the SID11 bit refused |
| Microchip's `canfdspi` driver | not fetched: the mirrors found were gone.  Its rule is in the reference manual's Example 3-2, enable ECC and initialise the RAM before the mode is left, which `ram_init` follows |

Embassy and esp-hal hold no driver for this part.  The brief named a
`mcp2518fd` crate; the registry's crate for this family is `mcp2517`, which
also drives the MCP2518FD.

## Related packages

- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `CanBus` trait, `CanBusFrame`, the SPI and GPIO traits and the board handles.
- [devices-nv](https://novo-lang.org/packages/devices-nv) names the wiring record
  a board's device table builds and `AsyncDeviceReady`.
- [can-nv](https://novo-lang.org/packages/can-nv) holds the FD frame header
  `CanFdFrame` and the DLC tables this driver uses, and the frame queue, filters,
  error state machine and ISO-TP above the controllers.
- `mcp2515-nv` drives the classic MCP2515 on the same trait.

## Tests

`novo test tests` runs the module over a double of the part that answers the
SPI instruction set.  The double keeps a byte memory behind RESET, READ and
WRITE with the registers' reset values, the mode change that takes a few
reads, the oscillator that becomes ready after a few reads, a transmit queue,
a transmit FIFO, a receive FIFO and a transmit event FIFO with the head and tail
that move when UINC is written, and the status and user address registers those
produce.  It counts any RAM access that is not a whole word, and keeps the log
of the bytes the driver sent.  A second double of the link carries a fixed
number of transactions and refuses the rest, so each step of the bring-up, the
send, the receive and the filter write is checked to fail alone.

The suite checks the bit timing against the reference manual's example and the
table above, the exact bytes of every transaction of the bring-up, a classic
and an FD frame through the object layout, the receive path for both formats
and a 64-byte payload, the filters, the interrupt flags, the loopback end to
end and the awaitable pin.  The identifier 0x18DAF110 is worked out by hand in
the suite's header.  `bash tests/coverage.sh` merges the suites' line coverage
over `src/`.

**Hardware: NOT VERIFIED.**  See the top of this page.

## Licence

Apache-2.0.
