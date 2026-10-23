# HelloWorldS2

A simplified, cross-platform host project for running **Hello World** on **SmileBASIC / S2**, without requiring the Raspberry Pi Pico keyboard server.

---

## ⚠️ Risk Warning

**Running homebrew software, custom native code execution, and memory RPC tools on proprietary console hardware carries inherent risks.** Improper usage, incorrect memory writes, network configuration issues, or incompatible firmware states can cause system instability, crashes, system freezes, or unresponsiveness. 

We (Main developer and contributors) are not responsible for any damage to your S2, PC, or other hardware; data loss; corrupted saves; or any other consequences arising from the use of this software. Always back up important data before making modifications. This software is provided as-is, without warranties of any kind. Use at your own risk and discretion.

---

## Overview & Version Support

- **Supported Environment:** This software has been proven and validated exclusively on **HOS 23.0.1** and **SmileBASIC 4.4.9**. No other versions have been validated.
- **Platform Compatibility:** Primarily tested and developed on **Windows**, with community bug-fix corrections integrated to improve compatibility across both **Windows** and **macOS**.
- **Host PC:** Hosts the RPC bridge server, handles console qualification, performs module inspections, and runs the Hello World GPU app.
- **Console (S2):** You manually set up the bootstrap script and execute the connection sequence in SmileBASIC.

---

## Prerequisites & Dependency Explanations

1. **Node.js (v22+)**
   - *Why:* Serves as the core host runtime environment. It executes `server.js`, manages the TCP connection listener (`node:net`), hosts the local bridge HTTP server (`node:http`), and orchestrates the inspection and pipeline scripts.

2. **Python 3**
   - *Why:* Used to execute the asset rasterization script (`catalog-assets.py`) during menu preparation.

3. **Pillow (`PIL` - Python Imaging Library)**
   - *Why:* Used by `catalog-assets.py` to process homebrew application assets. It opens app icons, resizes them (`ImageOps.contain`), rasterizes application titles and descriptions into text bitmap tiles (`ImageDraw.Draw`), and packs them into a fixed GPU texture atlas (`catalog.bin`) so the S2 console can correctly render app thumbnails and menu labels in the homebrew interface. *(Note: `server.js` automatically checks for Pillow and installs it via `pip` if missing).*

4. **Network Access**
   - *Why:* Inbound TCP port **39380** must be open/allowed through your firewall so the S2 console can connect to the host PC's RPC server.

---

## Console Bootstrap Setup (Required)

Before connecting from your PC, you must set up the console-side RPC server script:
1. Retrieve the content of `bootstrap.bas` from the [ns2jb repository](https://gitlab.com/pepsipu/ns2jb/-/tree/main?ref_type=heads).
2. Save/load it into SmileBASIC on your console as `TXT:NS2` on the default workspace. This script acts as the core console-side RPC bridge handler.

---

## Getting Started

1. Start the RPC server on your host PC:
   ```cmd
   node server.js
   ```
   *(Optional Arguments & Flags)*:
   - `[private-ip]` (Optional positional argument): Specify your host's local private IP address directly (e.g. `node server.js 192.168.1.50`). If omitted, the server will automatically detect available network interfaces or prompt you to select one.
   - `--verbose` or `--debug`: Enable low-level RPC and native exchange logs.
   - `--pairingCode <n1,n2,n3,n4>` (or `-p`): Specify a custom 4-part numeric pairing code (e.g. `node server.js --pairingCode 1234,5678,9012,3456`).  
   
   > **Note on Pairing Code:** By default, the pairing code is set to `[0, 0, 0, 0]`. We default this because we expect this software to be run on a **local private network**, not on public Wi-Fi where anyone can reach your host. However, if you prefer to define your own custom pairing code, you can supply it via `--pairingCode`, and it will be automatically reflected in the generated `V=START(...)` connection command.

2. The script will display your computer's local IP address and the exact `V=START(...)` command for SmileBASIC.

3. On your S2 console running SmileBASIC Direct mode, execute the complete connection sequence:
   ```basic
   LOAD "TXT:NS2"
   RUN
   REMOVE NSR,&H60000000,&H3FFFFFFF
   ```
   *Line-by-line explanation:*
   - `LOAD "TXT:NS2"` — Loads the bootstrap script from text storage into the active SmileBASIC workspace.
   - `RUN` — Executes the bootstrap script, initializing network communication and establishing the console-side RPC server bridge.
   - `REMOVE NSR,&H60000000,&H3FFFFFFF` — Prepares and allocates memory regions (`NSR`, base address `&H60000000`, length `&H3FFFFFFF`) required for homebrew payload mapping and execution.

4. Run the generated `V=START(...)` command provided by the server. *(Note: Assigning the output to variable `V` via `V=` is required in SmileBASIC for functions with return values).*

5. The host will automatically detect the console connection, qualify the native session, run inspection steps, prepare the **Hello World** app, and render it on your console!

---

## Engineering Enhancements & Reliability Improvements

This repository builds upon upstream foundations with extensive engineering enhancements for stability, timeout management, and cross-platform execution:

- **RPC & Connection Layer Stability:** 
  - Increased timeouts across connection layers and RPC memory transfer operations to prevent dropped requests and session termination during heavy memory transfers.
  - Implemented real-time memory transfer progress logs and detailed diagnostics, fully guarded behind a clean `--verbose` flag so normal console output remains concise.
- **Native Calling & Exchange Reliability:** 
  - Upgraded native exchange handling (`native.cjs`) to prevent event-loop deadlocks.
  - Added firmware instruction mismatch tolerance in `verifyFunction`: minor native instruction byte variations across different console firmware versions are now gracefully tolerated (issuing a warning instead of hard-crashing).
- **Frame Loop & Pipeline Hardening:** 
  - Replaced custom manual render loops with HBLauncher's continuous `launcher-loop.cjs` process to properly drive frame submission and presentation.
  - Bypassed external toolchain dependencies (`spirv-val`) when validating app manifests (`catalogTools.readApp`), preventing startup failures on host machines lacking toolchain binaries.
- **Cross-Platform Python & Font Fallbacks:** 
  - Added multi-alias Python support (`python3`, `python`, `py`) with automatic Pillow dependency checking and installation via `pip`.
  - Implemented multi-path system font resolution across Windows, macOS, and Linux to ensure robust catalog asset rasterization.
- **Robust Input Handling:** 
  - Added automatic fallback stubs for unresolved HID input state imports (`GetNpadState`, `GetTouchScreenState`) across differing system software versions.

---

## Credits & Acknowledgements

- [HBLauncher by ravensclawx](https://github.com/ravensclawx/HBLauncher) — Homebrew launcher architecture and GPU rendering pipeline.
- [ns2jb by pepsipu](https://gitlab.com/pepsipu/ns2jb/-/tree/main?ref_type=heads) — Initial deployment code, bootstrap script, RPC foundations, and Hello World app structure.
