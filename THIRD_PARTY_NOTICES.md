# Third-Party Notices

This repository contains original RA-08 AT modem firmware plus selected
third-party SDK components required to build for the Ai-Thinker RA-08 /
ASR6601 platform.

The root MIT license applies only to original project files unless a file says
otherwise. It does not relicense third-party files. Keep upstream notices in
place when copying, modifying, or redistributing these components.

Known third-party components in the current tree:

- Ai-Thinker / ASR6601 SDK components under `build/`, `drivers/`, `platform/`,
  and parts of `lora/`. The imported SDK did not include a single root license
  file in this checkout; retain in-file notices and review upstream terms before
  publishing binaries or source redistributions.
- ARM CMSIS headers under `platform/CMSIS/`, carrying ARM copyright and
  BSD-style redistribution terms in file headers.
- Semtech/SX126x radio support and LoRa utility code under `lora/radio/`,
  `lora/driver/`, and `lora/system/`, with file headers referring to a
  Revised BSD License where applicable.
- Marco Paland `printf` implementation in `platform/system/printf-stdarg.c`,
  licensed under MIT in its file header.
- CMAC crypto code under `lora/system/crypto/`, carrying its own permissive
  license notices in file headers.
- STMicroelectronics-derived timer code in `lora/system/timer.c`, whose header
  includes ST-specific redistribution and execution restrictions. Review this
  file carefully before public redistribution or replace it with a clean
  implementation.

Vendor datasheet PDFs are intentionally not committed by default. See
`Datasheets/README.md`.
