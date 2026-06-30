# RA-08 / ASR6601 Technical Plan

## Scope

Initial target: prepare and implement robust firmware for Ai-Thinker RA-08 so the module behaves as a stand-alone UART radio modem and answers AT commands directly from a PC/USB-serial host.

## Sources Checked

- `Datasheets/ra-08_v1-1-0_specification.pdf`
- `Datasheets/ra-08-kit.pdf`
- Public Ai-Thinker SDK cloned in this workspace:
  `Ai-Thinker-LoRaWAN-Ra-08`
- SDK clone commit:
  `5fdef9a2dc83ee9b089171ed6f98c8c24915e351`
  (`2024-02-21`, "config rf switch for single_tone test")
- Upstream repository:
  https://github.com/Ai-Thinker-Open/Ai-Thinker-LoRaWAN-Ra-08
- Ai-Thinker documentation entry:
  https://docs.ai-thinker.com/en/Ra-08/index.html

## Architecture Confirmation

RA-08 must be treated as an LPWAN SoC module, not as an SPI LoRa transceiver.

The RA-08 uses the ASR6601. The datasheet describes it as an LPWAN wireless communication SoC integrating:

- RF transceiver
- modem
- 32-bit MCU / ARM core
- LoRa and FSK-family modulation support

The Ai-Thinker SDK README identifies the specific module SoC as ASR6601CB with:

- 32-bit 48 MHz ARM Cortex-M4 core
- 128 KB flash
- 16 KB SRAM

Implication: an external controller or PC must not drive RA-08 like SX1278/SX1262 over SPI. Instead, the RA-08 runs its own ASR6601 firmware and exposes a high-level UART AT protocol.

The SDK contains files named `sx126x` and `radio`, but these are the ASR6601 internal radio abstraction used inside the ASR firmware. They do not mean an external host can access an external SX126x over SPI.

## Hardware Notes

RA-08 module:

- Logic and IO level: 3.3 V.
- Supply range: 2.7 V to 3.6 V.
- Recommended supply: 3.3 V, peak/current budget >= 500 mA.
- Frequency band for RA-08: 410 MHz to 525 MHz.
- RA-08H is the 803 MHz to 930 MHz variant, not the same RF band.
- External antenna required.

Relevant RA-08 pins:

- `VCC`: 3.3 V supply.
- `GND`: common ground.
- `TXD`: GPIO17 / UART TX.
- `RXD0`: GPIO16 / UART RX, documented as burning/programming port.
- `LPRXD`: GPIO60 / LPUART RX, documented as communication serial port.
- `IO2`: GPIO2 / BOOT.
- `RST`: active-low reset.
- `SWCLK`: GPIO7 / SWD_CLK.
- `SWDIO`: GPIO6 / SWD_DATA.

Boot mode:

- IO2 default low: normal SPI/flash startup.
- IO2 high during reset: download mode.

Runtime UART recommendation:

- For a clean product design, expose `TXD`, `LPRXD`, `RXD0`, `IO2`, `RST`, `SWCLK`, `SWDIO`, `VCC`, and `GND` on test pads or a small programming header.
- For the current stand-alone platform, the PC/USB-serial adapter talks directly to the RA-08 AT UART.
- Keep `RXD0` available for serial flashing. If UART0 (`TXD`/`RXD0`) is used for application AT commands during development, isolate it with jumpers or series resistors so the runtime serial host and external programmer cannot fight each other.
- Add 10-100 ohm series resistors on signal lines if the board layout allows it, matching the datasheet guidance.

## Existing SDK / Firmware Findings

The public Ai-Thinker SDK is usable as the base.

Important examples:

- `projects/ASR6601CB-EVAL/examples/lora/pingpong`
  Basic peer-to-peer LoRa example.
- `projects/ASR6601CB-EVAL/examples/lora/lora_test`
  Fixed-frequency LoRa test firmware with AT-like commands:
  `AT+CTX`, `AT+CRX`, `AT+CTXCW`, `AT+CSLEEP`, `AT+CSTDBY`.
- `projects/ASR6601CB-EVAL/examples/lora/lora_factory_test`
  Factory-test firmware with a simple AT parser, config stored in flash, and RF parameter validation.
- `projects/ASR6601CB-EVAL/examples/lorawan/lorawan_at`
  LoRaWAN AT firmware, not directly the desired P2P UART modem.
- `projects/ASR6601CB-EVAL/examples/ota`
  OTA bootloader/app examples.

Important libraries/drivers:

- `drivers/peripheral`: UART, GPIO, flash, power, RCC, timers, etc.
- `lora/radio`: high-level radio API.
- `lora/radio/sx126x`: internal ASR/SX126x-style radio implementation.
- `lora/mac`: LoRaWAN stack.
- `lora/linkwan`: LinkWAN/AT related code.

Conclusion:

- There is factory/default AT firmware according to the SDK README and Ai-Thinker docs, but the local RA-08 datasheet does not document the full command set needed here.
- Existing AT examples are useful references, not the final interface.
- For our requirement, implement our own firmware and AT command surface.

## Toolchain

Official/SDK flow is a bare-metal Makefile build, not ESP-IDF and not PlatformIO.

Required tools:

- Linux or WSL/Ubuntu is recommended.
- `gcc-arm-none-eabi`
- `make`
- `git`
- Python 3
- Python packages: `pyserial`, `configparser`

SDK README uses:

```sh
sudo apt-get install gcc-arm-none-eabi git vim python python-pip
pip install pyserial configparser
```

Modern WSL/Ubuntu equivalent:

```sh
sudo apt update
sudo apt install -y gcc-arm-none-eabi make git python3 python3-pip
python3 -m pip install pyserial configparser
```

Build environment:

```sh
cd /mnt/d/Documente/RA08/Ai-Thinker-LoRaWAN-Ra-08
source build/envsetup.sh
cd projects/ASR6601CB-EVAL/examples/lora/pingpong
make
```

PowerShell status in this workspace:

- `make` was not found in PATH.
- `arm-none-eabi-gcc` was not found in PATH.
- Build was not run locally from PowerShell.

## Flashing

The SDK flashes through `build/scripts/tremo_loader.py`.

Default Makefile settings:

- serial port: `/dev/ttyUSB0`
- baud: `921600`
- flash address: `0x08000000`

Example:

```sh
python3 build/scripts/tremo_loader.py -p /dev/ttyUSB0 -b 921600 flash 0x08000000 out/firmware.bin
```

The script can toggle boot pins if the USB-UART adapter exposes the lines:

- DTR: GPIO2 / BOOT
- RTS: reset

Manual download mode if DTR/RTS are not wired:

1. Drive IO2 high.
2. Toggle/reset RA-08 with `RST`.
3. Run `tremo_loader.py ... flash 0x08000000 ...`.
4. Return IO2 low.
5. Reset module to boot application firmware.

The SDK also includes:

- `tools/programmer/ASR6601Programmer_v0.9.exe`
- `tools/programmer/TremoProgrammer_v0.8.exe`
- `tools/FLM/ASR6601.FLM`
- Keil project generation scripts

## Debugging

Preferred initial debug:

- UART logs on the same or a separate UART during bring-up.
- Two RA-08 boards for TX/RX verification.
- Logic analyzer on the PC/USB-serial-to-RA08 UART lines when debugging framing or boot timing.

SWD:

- RA-08 exposes SWD pins (`SWCLK`, `SWDIO`).
- The SDK includes an ASR6601 FLM flash algorithm and Keil project support.
- Keil + compatible probe is the most plausible vendor-supported debug path.
- OpenOCD support was not confirmed from the SDK; do not assume it until tested.

## Proposed Firmware Project

Create a new ASR6601 firmware target under the SDK, for example:

```text
projects/ASR6601CB-EVAL/examples/ra08_uart_modem/
  Makefile
  cfg/gcc.ld
  inc/
    app_config.h
    at_parser.h
    board_ra08.h
    cfg_store.h
    modem_protocol.h
    radio_service.h
    uart_transport.h
  src/
    main.c
    board_ra08.c
    uart_transport.c
    at_parser.c
    cfg_store.c
    radio_service.c
    sleep_mgr.c
    commands_basic.c
    commands_radio.c
    commands_power.c
```

Use the existing `lora_test` example as the radio base, and the `lora_factory_test` example as the command/config-storage reference. Do not include the LoRaWAN stack for the first P2P modem firmware; 128 KB flash and 16 KB SRAM are tight.

## Runtime Design

Main loop:

1. Initialize clocks, GPIO, UART, flash config, radio, watchdog.
2. Start UART RX interrupt/DMA/ring buffer.
3. Parse CR/LF terminated AT lines.
4. Dispatch commands into a nonblocking modem state machine.
5. Call `Radio.IrqProcess()` continuously.
6. Emit synchronous responses (`OK`, `ERROR`) and asynchronous URCs.

UART transport:

- Default: 115200 8N1 for development.
- Optional later: configurable baud.
- RX ring buffer, line parser, overflow detection.
- Never block indefinitely inside a command handler.

Radio service:

- Own all `Radio.*` calls.
- Maintain state: `IDLE`, `RX`, `TX`, `SLEEP`, `ERROR`.
- Handle TxDone, RxDone, TxTimeout, RxTimeout, RxError callbacks.
- If `AT+SEND` is called while RX is on: stop RX/standby, TX, then restore RX if it was enabled.

Config storage:

- Reserve final flash page at `0x0801F000` for config, matching the factory-test reference.
- Adjust linker script so application code does not use the final 4 KB.
- Store `{magic, version, length, sequence, config, crc32}`.
- Prefer two slots within the 4 KB page if page/erase constraints allow it.
- On CRC/version failure, load defaults and report recoverable config reset.

Suggested defaults:

- Frequency: 433000000 Hz for RA-08 development, unless a legal/regional channel plan is specified.
- Channel table: small fixed table in the 410-525 MHz RA-08 band.
- TX power: conservative default such as 14 dBm; allow up to SDK-supported 22 dBm with validation and regulatory warning.
- SF: 7 by default.
- BW: 125 kHz by default.
- CR: 4/5 by default.
- Preamble: 8.
- CRC: on.
- Explicit header.
- IQ inversion: off.

## AT Command Surface

Line ending: accept `\r`, `\n`, or `\r\n`.

Responses:

- Success: `OK`
- Error: `#ERROR: <reason>`
- Async RX: `+RX:<rssi>,<snr>,<len>,<hex>`
  - `rssi`: signed RSSI in dBm, reported by the radio driver.
  - `snr`: signed SNR in dB, reported by the radio driver.
  - `len`: payload length in bytes.
  - `hex`: received payload encoded as uppercase hex.
- Async TX done: `+TXDONE`
- Async TX timeout: `+TXTIMEOUT`
- Async RX error: `+RXERROR`

Initial commands:

```text
AT
AT?
AT+?
AT+HELP
AT+VERSION?
AT+CFG?
AT+STATUS?
AT+DEFAULT
AT+FREQ?
AT+FREQ=<Hz>
AT+CHAN?
AT+CHAN=<n>
AT+PWR?
AT+PWR=<dBm>
AT+SF?
AT+SF=<5..12>
AT+BW?
AT+BW=<bandwidth>
AT+CR?
AT+CR=<coding_rate>
AT+RX?
AT+RX=ON
AT+RX=OFF
AT+SEND=<data>
AT+LASTPKT?
AT+SLEEP?
AT+SLEEP
AT+WAKE
```

Recommended `AT+SEND=<data>` format:

- Make `<data>` hex by default to avoid ambiguity with commas, CR/LF, and binary bytes.
- Example: `AT+SEND=48656C6C6F`
- Later optional extension: `AT+SENDSTR="Hello"`.

Validation:

- `AT+FREQ=<Hz>`: for RA-08, reject values outside 410000000-525000000.
- `AT+CHAN=<n>`: index into configured channel table.
- `CHAN=-1`: manual frequency mode, set by `AT+FREQ=<Hz>`. `AT+CFG?` keeps this numeric value for script parsing; `AT+CHAN?` reports `+CHAN:-1 (manual frequency)` for humans.
- `AT+PWR=<dBm>`: validate against the Ai-Thinker factory-test range, currently 2-22 dBm.
- `AT+SF=<5..12>`: SDK/radio supports SF5-SF12, but examples often use SF7-SF12; SF5/SF6 need explicit validation on real hardware.
- `AT+BW=<bandwidth>`: accept numeric Hz (`125000`, `250000`, `500000`) and optionally SDK indices (`0`, `1`, `2`).
- `AT+CR=<coding_rate>`: accept `4/5`, `4/6`, `4/7`, `4/8` or SDK indices `1..4`.
- `AT+SEND=<hex>`: require compact uppercase/lowercase hex only, even number of hex digits, 1..255 bytes.

Sleep behavior:

- `AT+SLEEP`: reply `OK`, stop RX, and put the radio in sleep. The current firmware keeps UART command handling alive.
- `AT+WAKE`: only works as a UART command if the selected sleep mode supports UART/LPUART wake. For deeper sleep, wake must be a pin/reset event, then the firmware can print `+WAKE`.
- Radio-active operations while sleeping, such as `AT+SEND=...`, `AT+RX=ON`, and RF parameter writes, return `#ERROR: RADIO_SLEEPING (send AT+WAKE)`.
- Define this explicitly before relying on deep sleep current numbers.

## Robustness Requirements

- Watchdog enabled after bring-up.
- Parser fuzz tests on host if possible.
- RX buffer overflow reported, not silent.
- Max payload limit exposed by `AT+CFG?`.
- No malloc in normal command path if avoidable.
- Radio state guarded so command handlers cannot race callbacks.
- Config CRC and versioning.
- Build artifacts include `.elf`, `.bin`, `.map`, and size report.
- Keep a debug build with verbose logs and a release build with minimal logs.

## Test Plan

Bring-up:

1. Build and flash unmodified `lora/pingpong`.
2. Flash two RA-08 boards and confirm PING/PONG.
3. Flash `lora/lora_test` and confirm `AT+CTX` / `AT+CRX` behavior.

New firmware tests:

1. `AT`, `AT+HELP`, `AT+CFG?` smoke test.
2. Set each RF parameter and verify persistence across reset.
3. Two-module TX/RX test with `AT+RX=ON` on one side and `AT+SEND=...` on the other.
4. Boundary tests for invalid frequency, SF, BW, CR, payload length.
5. Power-cycle and reset recovery.
6. Sleep/wake tests with measured current.
7. Long-run RX test for missed packets and UART buffer stability.

## Key Risks / Open Checks

- Confirm exact runtime UART wiring for the target board: `LPRXD` vs `RXD0`.
- Confirm whether the final stand-alone board should expose IO2/RST for external flashing, or whether programming remains via the current RA-08-Kit/USB serial path.
- Confirm SWD probe/tool support before committing to SWD debug workflow.
- Confirm legal regional TX power/channel plan for the actual deployment country.
- Confirm SF5/SF6 behavior on RA-08 hardware; examples mostly use SF7-SF12 even though the datasheet lists SF5-SF12.
- Keep final firmware size under 124 KB if reserving the final 4 KB flash page for config.

## Recommended Next Step

Create the `ra08_uart_modem` firmware target inside the cloned SDK, derived from `lora_test`, with:

- ring-buffer UART transport
- table-driven AT parser
- persistent config
- radio service abstraction
- the initial command set listed above

For this platform there is no ESP32 integration step. The immediate follow-up is hardware RF validation with one or two RA-08 modules and an antenna connected.
