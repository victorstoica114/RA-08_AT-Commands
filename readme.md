# RA-08 UART Modem Firmware

Minimal ASR6601/RA-08 firmware that turns an Ai-Thinker RA-08 module into a
stand-alone LoRa P2P modem controlled with AT commands over UART.

RA-08 is an LPWAN SoC with its own MCU and radio. It is not an SPI transceiver
driven by an ESP32.

## Layout

- `main.cpp` - board clock, UART, RTC, and modem loop entry point.
- `projects/ASR6601CB-EVAL/examples/ra08_uart_modem` - AT parser, radio modem
  state machine, flash config, linker script, and interrupt handlers.
- `platform`, `drivers`, `lora`, `build` - SDK components required by the
  modem firmware build.
- `docs/RA08_TECHNICAL_PLAN.md` - architecture, command set, validation notes,
  and flashing/debug details.
- `tools/tests/ra08_regression.py` - serial regression test for two RA-08
  modules on `COM20` and `COM21`.
- `Datasheets/README.md` - vendor datasheet references and redistribution
  notes. Keep PDF copies local unless Ai-Thinker grants redistribution rights.

## Build

From Git Bash or a shell where `mingw32-make` is available:

```sh
mingw32-make
```

The local toolchain path defaults to:

```text
tools/toolchain/bin/
```

Override it if you use a system-installed ARM GCC:

```sh
mingw32-make TOOLCHAIN_PATH=/path/to/arm-none-eabi/bin/
```

The output firmware is generated under:

```text
projects/ASR6601CB-EVAL/examples/ra08_uart_modem/out/
```

## Flash

Put the module in download mode if automatic DTR/RTS reset does not work:

1. Hold `BOOT` / `IO2`.
2. Tap `RESET`.
3. Release `RESET`.
4. Release `BOOT` / `IO2`.

Then flash:

```sh
mingw32-make flash SERIAL_PORT=COM20
```

For the second module:

```sh
mingw32-make flash SERIAL_PORT=COM21
```

## Test

With two modules connected on `COM20` and `COM21`:

```sh
python tools/tests/ra08_regression.py
```

The regression covers basic AT commands, error formatting, sleep guards,
manual-frequency mode (`CHAN=-1`), channel mode, and bidirectional radio
traffic.

## AT Commands

Commands are sent over UART at `115200 8N1` and are terminated with CRLF.
Successful commands return `OK`; errors use the `#ERROR: <reason>` format.

Basic commands:

```text
AT
AT?
AT+?
AT+HELP
AT+VERSION?
AT+CFG?
AT+STATUS?
AT+DEFAULT
```

Radio configuration:

```text
AT+FREQ?
AT+FREQ=<410000000..525000000>
AT+CHAN?
AT+CHAN=<n>
AT+PWR?
AT+PWR=<2..22>
AT+SF?
AT+SF=<5..12>
AT+BW?
AT+BW=<0|1|2|125000|250000|500000>
AT+CR?
AT+CR=<1..4|4/5..4/8>
```

Radio operation and power state:

```text
AT+RX?
AT+RX=ON
AT+RX=OFF
AT+SEND=<hex>
AT+LASTPKT?
AT+SLEEP?
AT+SLEEP
AT+WAKE
```

Useful response formats:

```text
+CFG:FREQ=<hz>,CHAN=<n>,PWR=<dbm>,SF=<sf>,BW=<hz>,CR=<rate>,RX=<ON|OFF>,SLEEP=<0|1>,MAXPL=<bytes>
+STATUS:VERSION=<ver>,FREQ=<hz>,CHAN=<n>,PWR=<dbm>,SF=<sf>,BW=<hz>,CR=<rate>,RX=<ON|OFF>,SLEEP=<0|1>,TXBUSY=<0|1>,LASTPKT=<0|1>
+RX:<rssi>,<snr>,<len>,<hex>
+LASTPKT:<rssi>,<snr>,<len>,<hex>
```

Notes:

- `AT?` and `AT+?` are aliases for `AT+HELP`.
- `AT+DEFAULT` restores the safe default config, disables RX, and wakes the
  radio if it was sleeping.
- `CHAN=-1` means manual frequency mode set with `AT+FREQ=<Hz>`.
- `AT+SEND=<hex>` accepts uppercase or lowercase hex, even length only, up to
  `MAXPL` bytes.
- Radio-changing commands and TX/RX start commands are guarded while sleeping
  and return `#ERROR: RADIO_SLEEPING (send AT+WAKE)`.

## Default Working Config

```text
FREQ=433000000
CHAN=0
PWR=14
SF=7
BW=125000
CR=4/5
RX=OFF
SLEEP=0
```

`CHAN=-1` means manual frequency mode set with `AT+FREQ=<Hz>`.

## License

Original RA-08 AT Commands project code, documentation, tests, and build glue
are licensed under MIT; see `LICENSE`.

Bundled SDK/vendor files keep their original notices and are not relicensed by
this project. See `THIRD_PARTY_NOTICES.md` before publishing or redistributing
the full source tree.
