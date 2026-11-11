# Changelog

## 0.1.0

The first release: a driver for Microchip's MCP2518FD external CAN FD
controller on an SPI bus.

- `Mcp2518`, the type a board's device table builds with the board's
  oscillator, implementing embedded-hal-nv's `CanBus` for classic frames, with
  its own methods for FD frames, the filters, the transmit event FIFO, the
  interrupt flags, the GPIO pins and register access.
- The SPI instructions RESET, READ and WRITE as functions over `FdLink`, a
  trait that a type holding a bus implements, and the framing as functions over
  `SpiBus` and `GpioOut`.  The buffers are `Vec[u8; 96]` held by the caller.
- The bring-up: the bus opened at 17/40 of the oscillator and raised once the
  oscillator is ready, Configuration mode requested with ABAT, RESET confirmed
  by CiCON reading its reset value, the oscillator and PLL started and polled,
  IOCON written a byte at a time, ECC enabled and the whole message RAM written
  with zeros, the interrupts enabled, and the mode requested and confirmed.
- The bit timing for the nominal and data phases found from the system clock,
  the rate and the sample point, with the table for 20 and 40 MHz at 500 kbit/s
  and 1 Mbit/s nominal and 2 and 5 Mbit/s data, automatic transmitter delay
  compensation for a data phase of 1 Mbit/s and up, and the rate read back from
  the part.
- A fixed message RAM layout: the transmit event FIFO, the transmit queue, FIFO
  1 as the receive FIFO and FIFO 2 as a transmit FIFO, 1792 of 2048 bytes, with
  the message objects built and decoded as T0, T1, R0 and R1 words.
- Classic and FD frames sent through the queue or the FIFO with a sequence
  number, received through the user address walk and UINC, and the transmit
  events read back.
- Acceptance filters written disabled, read back and enabled; the interrupt
  flags, their enables and their clearing; the error counters and state; the
  GPIO pins; the ECC status and the device ID register.
- `mcp2518fd_irq.Mcp2518Line`, the INT pin as devices-nv's `AsyncDeviceReady`.
- A device program, `examples/can_fd_echo.nv`, and `tests/device_build.sh`,
  which builds it for the nRF52840 DK and the nRF54LM20 DK and runs the
  emulated board's image under QEMU.
- The suite runs over a double of the part and covers every line of `src/`;
  `tests/coverage.sh` measures it.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an MCP2518FD.
- The data sheet read is DS20006027B (December 2020) and the reference manual
  DS20005678E.  The data sheet gives no value for the DEVID register, so the
  driver reads it and tests nothing, and has no `Identifiable`.
- The SPI CRC instructions, sleep and wake-up, time stamping, the bus diagnostic
  registers, other RAM layouts, FIFOs 3 to 31, DeviceNet filtering and the
  transmit bandwidth sharing are not implemented.
- The trait's `open` runs Normal CAN 2.0 mode; a node on a bus with FD frames
  opens with `MCP_MODE_NORMAL_FD` through `open_with`.
- `receive` on the trait drops an FD frame.
- A full transmit event FIFO discards its next event (TEFOVIF) and does not
  stop transmission.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive SPI with SPIM2 and SPIM21: 19,540 and 20,988 bytes of flash.  A reply
  from the part over either bus is NOT VERIFIED.
- `holds` compares a register's read-back through a function of the optional
  and the value, because `reg_read(...) != Some(obj)` with `obj` a local leaks
  one object per call
  (`a-comparison-of-an-optional-with-some-of-a-local-leaks-one-object-when-the-callee-holds-a-vec`).
- Microchip's `canfdspi` driver could not be fetched; its rule on ECC and the
  RAM is taken from the reference manual's Example 3-2.
