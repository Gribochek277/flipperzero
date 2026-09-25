> Sanitized mirror of Forgejo `serhii/flipperzero`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/flipperzero` | Synced: 2026-10-05T01:38:27Z

---

# flipperzero

Pi-agent extension for controlling a Flipper Zero over USB serial.

- **No separate LLM, no MCP** — pi is the brain; the extension only exposes
  tools that speak the Flipper serial CLI protocol.
- Reference (do not copy): https://github.com/xunholy/promptzero (AGPL, used as architecture reference only)
- Protocol reference (verified live on FW 1.4.3): [docs/protocol.md](docs/protocol.md)

## Layout

- `extension/serial.ts` — protocol-pluggable serial engine (`SerialSession`): blocking fd + single
  thread-pool read, prompt tracking, framing, abort, resync, atomic
  subshells, port autodiscovery. `FlipperSerial` pins the Flipper
  profile for API compatibility.
- `extension/profiles.ts` — `ProtocolProfile` config: prompt pattern,
  line terminator, abort byte, baud, echo shape, port discovery,
  command guard. Shipped profiles: **flipper** (230400, `>:`/`[nfc]>:`
  prompts, \r-lines, `0x03` abort) and **marauder** (115200,
  `> ` prompt without trailing newline, LF-lines, no abort byte —
  `stop` is a command there).
- `extension/guard.ts` — central command deny list (see Security);
  wired in as the Flipper profile's `assertCommand`.
- `extension/storage-validate.ts` — shared path/content validation for the
  storage-touching tools (text only for writes, /ext|/int, size cap);
  standalone, no hardware.
- `extension/tools.ts` — flipper_* tools: read-only/receive-only +
  confirm-gated `flipper_write`, `flipper_ir_tx`, `flipper_subghz_tx_file`,
  `flipper_nfc_emulate`, `flipper_loader_open`; all `defineTool`,
  sequential; refuse with "Flipper suspended, bridge active" while the
  Marauder bridge runs.
- `extension/marauder-bridge.ts` — the Marauder-over-Flipper bridge (ticket
  #9): raw Marauder probe (blocking pty read + timeout race), version
  extraction from boot banners, bridge state machine.
- `extension/marauder-tools.ts` — `marauder_bridge_open` / `marauder_bridge_close`
  / `marauder_status` + the active-operation tools (ticket #11):
  `marauder_scan_ap` / `marauder_sniff` (confirm-gated) / `marauder_stop` /
  `marauder_save` / `marauder_list`. All run over any active transport
  (bridge or direct USB) with the client-side pre-stop rule (a `stopscan`
  before every start) and the engine's `drain()` capture window.
- `extension/index.ts` — entry point (serial + bridge + both tool sets).
- `dev/test-serial.ts` — live-device test harness (`node dev/test-serial.ts`).
- `dev/test-guard.ts` — deny-list tests, no hardware required.
- `dev/test-marauder.ts` + `dev/fake_marauder.py` — Marauder profile
  tests against a pty-backed fake device (firmware-faithful), no
  hardware required.
- `dev/test-write.ts` — `flipper_write` tests: validators, live engine
  write/read-back, and the tool layer (confirm/decline/no-UI) with a mock
  ctx; needs the device for the live part.
- `dev/test-marauder-bridge.ts` — Marauder bridge tests (ticket #9) against
  pty fakes (fake Marauder + fake Flipper CLI), no hardware.
- `dev/test-marauder-ops.ts` — Marauder active-operation tests (ticket #11):
  drain(), scan_ap, sniff confirm paths (no UI / declined / approved),
  stop/save/list, wire-order checks, bridge mode — pty fake, no hardware.
- `dev/demo-marauder-bridge.ts` — LIVE bridge walk (needs Flipper + Marauder
  + a user on the Flipper buttons): open GPIO → user starts USB-UART Bridge
  → status (version/help/settings) → release.
- `dev/demo-marauder-ops.ts` — LIVE ops walk (ticket #11, needs the same
  setup + user confirmations on the terminal): scan 20 s → sniff 15 s
  (default pmkid; `node dev/demo-marauder-ops.ts deauth` for the attack-class
  dialog) → stop → list → save → release.
- `dev/demo-marauder-direct.ts` — LIVE direct-USB status (ticket #10):
  discovers a Marauder on its own USB (PI_MARAUDER_PORT / by-id bridge
  chips / ttyUSB*/ttyACM*), probe-validates the CLI, prints the status;
  on a board without a serial port it demonstrates the clear guidance.
- `dev/demo-aio-subghz.ts` — LIVE AIO board Sub-GHz walk (ticket #12, needs
  the board + a user for the switches: MODE -> BLUE/CC1101, RX-TX -> RX):
  hosts a stock `subghz rx <freq> 1` (CC1101_EXT, the board's high-gain
  path) for 20 s and diagnoses the external-vs-internal fallback from the
  `device:` line; `--nrf24` adds the guided nRF24 launch (community app;
  no stock driver). RX-only.

## Tools

All tools are `executionMode: "sequential"` (the serial port is a shared
resource). Every tool goes through the engine's single choke point
(`doSend`), which runs the command guard BEFORE the port is opened.

`flipper_status`, `flipper_help`, `flipper_sd_list`, `flipper_sd_tree`,
`flipper_sd_read`, `flipper_ir_rx`, `flipper_subghz_rx`, `flipper_nfc_scan`,
`flipper_rfid_read`, `flipper_ikey_read`, `flipper_gpio_read` — read-only /
receive-only.

`flipper_write` — the first write tool: text file to storage (create/append,
16 KiB cap, verbatim). Always asks the user via `ctx.ui.confirm` (path +
size in the dialog) and fails closed when no confirmation UI is available.

`flipper_ir_tx` — transmit an IR signal (a REAL signal): explicit
protocol/address/command (hex) or replay of a saved .ir file (parsed type).
Confirm-gated; the device re-validates protocol ranges (its error is
surfaced verbatim).

`flipper_subghz_tx_file` — replay a saved .sub file on the CC1101 radio
(a REAL transmission; the tool description carries the working bands
299-348 / 387-464 / 779-928 MHz). Confirm-gated with path + repeats.

`flipper_nfc_emulate` — act as a saved .nfc tag for a given duration
(nfc subshell, `emulate -f`; the engine's ETX abort stops it, guaranteed
return to the main shell). Confirm-gated with path + duration.

`flipper_loader_open` — launch a Flipper app by name (`loader open`). The
CLI stays alive; the device's UI switches to the app (some apps can only be
closed manually by the user). Confirm-gated with the app name; returns the
device's verbatim result (launched / not found).

Marauder tools (ticket #9, the Marauder speaks on the Flipper's USB port
via the stock USB-UART Bridge scene of the GPIO app):

`marauder_bridge_open` — confirm-gated: opens the GPIO app on the Flipper,
the user selects "USB-UART Bridge" on the Flipper screen (defaults: USB
Channel `0 (CLI)`, Baudrate `Host`, UART Pins `13,14`), the tool polls the
same USB port until the Marauder CLI appears. While the bridge is active
every `flipper_*` tool refuses with "Flipper suspended, bridge active".

`marauder_bridge_close` — verifies the Flipper CLI is back (the user exits
the USB-UART app on the Flipper via BACK, or re-plugs USB) and clears the
bridge state; otherwise returns the release instructions.

`marauder_status` — Marauder firmware version (from the boot banner;
when no banner was captured, a one-shot reboot captures it — there is no
runtime version command), the available command list (`help`) and current
`settings`. Transport: the active Flipper bridge, or direct USB (a
Marauder build with its own serial port — ttyUSB*/ttyACM* auto-discovered,
`PI_MARAUDER_PORT` overrides; ticket #10). AIO board V1.4: USB-C is the
ESP32-S2 native USB — if no port appears, the bridge is the path.
Clear errors when the port is missing or is not the Marauder CLI.

Marauder active operations (ticket #11, verified against v1.15.1 firmware
sources + live). The firmware runs these operations ASYNC: the start command
returns to the prompt immediately and results stream to the console until
`stopscan`. Each tool therefore sends `stopscan` first (client-side pre-stop
rule — idempotent), starts the operation, drains the console for the
requested window (`SerialSession.drain`, ticket #2's engine; without a
reader the tty buffer fills and output is lost), sends `stopscan` again and
reports the capture. All work over any active transport.

`marauder_scan_ap [durationSec]` — receive-only (no confirm): `scanall`
for the window (default 20 s), then the device's authoritative AP table
(`list -a`: `[i][CH:ch] essid rssi`) plus the live "new AP" lines captured
from the console. Leaves the device idle (post `stopscan`).

`marauder_sniff {mode, channel?, aggressive?, durationSec}` — CONFIRM-GATED
(fails closed without a confirmation UI): `sniffpmkid` (PMKID / 4-way
handshake), `sniffdeauth` (**captures** deauth frames — it only listens;
the dialog flags it as attack-class anyway), `sniffsae` (WPA3 commit,
passive), `sniffbeacon` (passive). The radio actively captures for the
window (default 30 s); returns the console capture (the firmware's periodic
stats block: `Mgmt/Data/Beacon/Probe/Deauth/EAPOL/Complete EAPOL/RSSI/
DEAUTH TX`, plus per-frame lines) and the auto-saved PCAP name
(`/eapol_<n>.pcap`, SavePCAP setting). **Optional `channel` (1-14)**
targets a specific AP's channel: `pmkid` passes it as `-c <channel>` (the
firmware's own flag, per its live `help`); the argument-less modes get
`channel -s <channel>` first. Omit it to capture on the radio's current
channel — after a scan the radio has hopped channels, so an untargeted
sniff is a toss-up. **`aggressive: true`** (pmkid only, requires an
explicit `channel`) adds the firmware's `-d`: the Marauder **transmits
deauthentication frames** so the AP's clients reconnect and a handshake can
actually be captured instead of waited for — its own attack-class dialog
naming the channel, and the report says whether deauths went out
(`DEAUTH TX:`). Only against a network you own or are authorized to test.
**v1.15.1 gate:** in that firmware `-d` alone only flips the scan mode —
the actual transmission is gated by the **ForcePMKID setting**
(`send_deauth = loadSetting<bool>(ForcePMKID)`; fixed in master). The tool
therefore enables ForcePMKID for the aggressive run and **restores it
afterwards** (stated in the report).

`marauder_stop` — `stopscan`: stops an active scan/sniff (safe, idempotent;
use after an interrupted capture). `marauder_save` — `save -a`: the device
saves its own AP list to its own SD (`/APs_0.log`, JSON) — device-generated
content, no confirm. `marauder_list [selected]` — `list -a` / `list -s`:
the device's AP table (indices usable with the firmware's select/attack
commands).

## Security

### Deny list (hard block)

Every command passes through `assertCommandAllowed()` in the engine's single
choke point (`doSend`) **before the port is opened**. These commands are never
sent to the device — by any current or future tool:

| Command | Why it's blocked |
|---|---|
| `factory_reset` | wipes all device settings |
| `storage format` | erases the storage filesystem |
| `crypto store_key` | stores a new secure-enclave key — firmware: "NON-REVERSABLE OPERATION"; losing the old key makes SD data undecryptable |
| `start_rpc_session` | switches the serial session to RPC (protobuf) mode and breaks the text CLI |

Matching is anchored to the command's leading tokens, so legitimate paths that
merely contain a denied word (`storage read /ext/factory_reset.txt`) pass
through. Covered by `dev/test-guard.ts` (30 checks, no hardware).

### Risk table (every tool, one glance)

Tiers: **low** — read-only / receive-only; **confirm** —
`ctx.ui.confirm` before the action, fails closed (throw) when no
confirmation UI is available; **block** — not exposed as tools at all,
the commands are denied by the guard (deny list above).

| Tool | Tier | Confirmation dialog | What it actually does |
|---|---|---|---|
| `flipper_status` `flipper_help` `flipper_sd_list` `flipper_sd_tree` `flipper_sd_read` | low | — | device state / CLI help / storage browsing + read |
| `flipper_ir_rx` `flipper_subghz_rx` `flipper_nfc_scan` `flipper_rfid_read` `flipper_ikey_read` `flipper_gpio_read` | low | — | receive/scan only — the radios only LISTEN |
| `flipper_write` | confirm | path + size | text file to storage (16 KiB cap, verbatim, read-back verified) |
| `flipper_ir_tx` | confirm | protocol/address/command | **a REAL IR signal** from the device's IR LED |
| `flipper_subghz_tx_file` | confirm | path + repeats | **a REAL Sub-GHz transmission** (replay of a .sub file) |
| `flipper_nfc_emulate` | confirm | path + duration | **RF emission** — the device acts as a saved NFC tag |
| `flipper_loader_open` | confirm | app name | launches an app on the device (UI switches; some apps close only manually) |
| `marauder_bridge_open` | confirm | bridge setup (the user must be at the Flipper screen) | opens the GPIO app on the Flipper; the user starts the USB-UART Bridge scene |
| `marauder_bridge_close` | low | — | verifies the Flipper CLI is back, clears the bridge state |
| `marauder_status` `marauder_list` `marauder_save` `marauder_stop` | low | — | version/help/settings, AP tables, the device saving its own SD, `stopscan` (idempotent) |
| `marauder_scan_ap` | low | — | the radio SCANS (receive-only), then the device's AP table |
| `marauder_sniff` | confirm | mode + channel + duration (+ attack-class dialog for `aggressive`) | pmkid/sae/beacon listen passively; `deauth` listens for deauth frames; **`aggressive: true` TRANSMITS deauths** (pmkid only, explicit channel required, own attack-class dialog) |
| (deny-listed commands) | block | — | `factory_reset`, `storage format`, `crypto store_key`, `start_rpc_session` — never sent, by any tool |

The confirm/decline/no-UI paths are covered without hardware by the E2E
harness (`dev/test-e2e.ts`, Phase A2) and `dev/test-tx.ts`; the live
transmit paths are exercised only in the ticket-approved live tests.

### BLE transport (status)

Spike verdict (ticket #3, FW 1.4.3): the Flipper exposes a **RPC (protobuf)
serial service over BLE GATT only — there is no text CLI over BLE**
(`docs/protocol.md` → "BLE serial"). A host-side limitation on this machine
(bluez CCCD subscribe not taking) blocks the live round-trip; the write
half is verified. The project is **USB-only** (ticket #13 closed as
wontfix): BLE is not a transport here. Prototype/verification:
`node dev/test-ble.ts` (exit 0 = full round-trip, 2 = write verified but
subscribe blocked by the host, 1 = infrastructure error). The E2E harness
runs it in Phase B5 only when a BT adapter is present.

### Marauder & the bridge

The Marauder (ESP32-S2 on the AIO board) speaks a second protocol profile
on the same engine core (`extension/profiles.ts`): 115200, `> ` prompt
without trailing newline, LF-lines, no abort byte (`stop` is a command).
Transports (ticket #9-#10):

| Transport | When | Notes |
|---|---|---|
| Flipper USB-UART Bridge (GPIO app scene) | **the path on the AIO board V1.4** | its USB-C is the ESP32-S2 native USB (no UART bridge chip) — the port never enumerates in normal operation |
| Direct USB (`PI_MARAUDER_PORT` / auto-discovery) | Marauder builds with a real USB-UART (CH340/CP2102) | same tools, same profile |

While the bridge is active every `flipper_*` tool refuses with "Flipper
suspended, bridge active" and the Marauder tools take over. Active
operations (scan/sniff) are ASYNC in the firmware — the engine's `drain()`
capture window + the client-side pre-stop rule (`stopscan` before every
start) are what make them reliable (`docs/protocol.md` → "Marauder active
operations").

### AIO board V1.4 (hardware)

3-in-1 expansion: ESP32-S2 (Marauder) + nRF24 (2.4 GHz) + high-gain CC1101
(sub-GHz). The MODE switch routes the Flipper SPI/power to ONE radio at a
time and drives the board LED: **RED = nRF24, GREEN = ESP32 (home
position), BLUE = CC1101**; the RX-TX switch is the CC1101 gain selector
(RX = LNA, TX = PA). Control verdicts (ticket #12, `docs/protocol.md` →
"AIO board RF modules"):

- **CC1101: FEASIBLE, LIVE-VERIFIED** — stock `subghz` CLI, `device 1`
  (CC1101_EXT) under host control: `subghz rx <freq> 1` (passive; the demo
  is RX-only), ETX stop, self-power-down. `dev/demo-aio-subghz.ts`.
- **nRF24: no stock driver** — community apps only (Flipper Lab catalog /
  `.fap`), UI-operated; the host can only `loader open/stop`.
- Gotcha: the Flipper's VCP can die silently after board switching (screen
  alive, CDC dead) — a genuine full reboot restores it; recovery ladder in
  the `aio-board` skill.

See Gitea issues `serhii/flipperzero` for the per-ticket history.

### Post-v2 fixes

- `marauder_sniff` **channel targeting** — the firmware only takes a
  channel for `sniffpmkid` (`-c <channel>`); the other modes now get an
  explicit `channel -s <channel>` first. Without it a capture lands on
  whatever channel the radio happened to be on. (+9 checks in
  `dev/test-marauder-ops.ts`; the pty fake now parses `-c`/`channel -s`.)
- `marauder_sniff` **`aggressive`** — exposes the firmware's `-d` flag
  (`sniffpmkid -d`), which is what makes a *forced* handshake capture
  possible: the Marauder transmits deauths so the AP's clients reconnect.
  Gated hard: pmkid only, an explicit `channel` is mandatory, its own
  attack-class dialog, and the report states whether deauths were sent.
  (+11 checks: the `-d` flag on the wire, non-pmkid rejected, no-channel
  rejected, declined → nothing sent.)
- **v1.15.1 requires the ForcePMKID setting for `-d` to transmit** —
  `WiFiScan.cpp: this->send_deauth = settings_obj.loadSetting<bool>(
  text_table4[5])` (verified against the v1.15.1 sources; master later made
  the explicit active-EAPOL modes win over the setting). Live run on the
  AIO board before the fix: start line said "with deauthentication" but the
  stats read `DEAUTH TX: FALSE`, `Deauth: 0` — nothing went out. The tool
  now enables ForcePMKID for the aggressive window and restores it
  afterwards (verified: the setting reads `false` again after the run).
  Live run after the fix: `DEAUTH TX: TRUE`, `Deauth: 1` (one deauth frame
  seen in the air) and a complete handshake in `/eapol_19.pcap`.

## Roadmap (v2 — tracked on Gitea)

- [x] #1 Guard: deny-list опасных команд — **done**
- [x] #2 Protocol-pluggable serial core — **done** (profiles.ts + SerialSession)
- [x] #3 BLE spike: text CLI vs RPC — **done, verdict: RPC (protobuf) only in FW 1.4.3, no text CLI over BLE** (`docs/protocol.md` → "BLE serial"; live round-trip blocked by a host-side CCCD subscribe issue — see ticket)
- [x] #4 `flipper_write` (text write, confirm) — **done**
- [x] #5 `flipper_ir_tx` (IR transmit, confirm) — **done**
- [x] #6 `flipper_subghz_tx_file` (Sub-GHz replay, confirm) — **done**
- [x] #7 `flipper_nfc_emulate` (NFC tag emulation, confirm) — **done**
- [x] #8 `flipper_loader_open` (open app, confirm) — **done**
- [x] #9 Marauder bridge via Flipper + `marauder_status` — **done** (verified live on Marauder v1.15.1; `docs/protocol.md` → "Marauder bridge")
- [x] #10 Marauder direct USB + AIO board 1.4 — **done** (AIO V1.4: USB-C is the ESP32-S2 native USB, no UART — its CLI is reachable through the Flipper bridge only; generic direct-USB discovery + CLI-probe status for builds with a real USB-UART; live `v1.15.1` over the bridge — `docs/protocol.md` → "Marauder direct USB + AIO board V1.4")
- [x] #11 Marauder scan/capture tools (confirm-gated) — **done** (scan_ap/sniff/stop/save/list over bridge + direct USB; sniff confirm-gated, deauth attack-class; pre-stop rule; `drain()` capture — `docs/protocol.md` → "Marauder active operations")
- [x] #12 nRF24/CC1101 spike — **done, verdict**: the board is the **AIO Board V1.4** (3-in-1: ESP32-S2 Marauder + nRF24 + high-gain CC1101); its MODE switch (RED=nRF24 / GREEN=ESP32 / BLUE=CC1101) routes the Flipper SPI/power to ONE radio at a time — the Marauder firmware has no nRF24/CC1101 drivers. **CC1101: FEASIBLE — stock `subghz` CLI (`… <0=INT|1=EXT>`), host-controllable — LIVE-VERIFIED (`device: 1`, 20 s window)** (prototype `dev/demo-aio-subghz.ts`, RX-only, gain switch RX). **nRF24: NO stock driver** — community app only (catalog / `.fap`), UI-operated; the host can only `loader open/stop`. Board hardware + switches: `docs/protocol.md` → "AIO board RF modules"
- [x] #13 Flipper BLE transport — **closed as wontfix** (project goes USB-only; FW 1.4.3 exposes RPC-only over BLE anyway — spike #3)
- [x] #14 E2E harness + risk docs — **done** (`dev/test-e2e.ts`: Phase A mocks — 5 fake-backed suites + tx-confirm mock-UI paths, no hardware; Phase B live — status, write/read round-trip, passive RX, bridge state, BLE status if adapter; `dev/test-aio-subghz.ts` — subghz rx contract vs fakes; README: risk table (low/confirm/block) + BLE/Marauder/bridge/AIO sections)

## Dev

### Audit reports: `docs/audits/`

A completed WiFi audit of the operator's own AP (capture method, handshake
proof, keyspaces + counts, verdict, recommendations) lives in
`docs/audits/2026-09-29-dom-wifi-audit.md`. It is **redacted** — masked MACs,
neighbouring networks anonymised, the recovered password never reproduced —
and the raw capture / hashes / potfiles stay out of git (`audit-tmp/`,
`*.pcap`, `*.22000` are ignored).

### Project skill: `.agents/skills/pcap-handshake-audit/`

The capture pipeline ends on the Marauder's microSD (`eapol_<n>.pcap`); the
analysis half — tshark, `hcxpcapngtool -o …hc22000`, `hashcat -m 22000` /
`aircrack-ng`, the `wpa-pwd` decryption — is a project skill that the agent
loads on its own (project skills live in `.agents/skills/`, discovered by pi
from the working directory). It carries the per-step completion criteria and
the guardrails (own/authorized networks, scrub neighbours' BSSIDs, WPA3 gives
nothing to `-m 22000`, `pocl` is required before hashcat runs at all).

```bash
node dev/test-e2e.ts         # E2E harness (ticket #14): Phase A mock suites + Phase B live device
                             #   (--require-device, --bridge for the user-driven cycle)
node dev/test-guard.ts       # deny-list tests, no hardware
node dev/test-marauder.ts    # Marauder profile vs fake device, no hardware
node dev/test-aio-subghz.ts  # subghz rx contract vs fake Flipper (device:1 / fallback), no hardware
node dev/test-write.ts       # flipper_write: validators + live write/read-back (device)
node dev/test-tx.ts          # tx tools (#5-#6+): validators + live tx (device)
node dev/test-ble.ts         # BLE spike prototype: protobuf System.Ping over GATT (ticket #3)
node dev/test-marauder-bridge.ts  # bridge + direct-USB tests, pty fakes, no hardware
node dev/test-marauder-ops.ts     # active-ops tests (scan/sniff/stop/save/list + channel targeting), pty fake, no hardware
node dev/sniff-ap.ts         # LIVE one-shot targeted sniff (thin CLI over marauder_sniff; passive only)
                             #   --channel 3 --duration 60 [--mode pmkid] [--label DOM] [--yes] [--dry-run]
node dev/demo-marauder-bridge.ts  # LIVE bridge walk (Flipper + Marauder + user on buttons)
node dev/demo-marauder-direct.ts  # LIVE direct-USB status (board on its own USB)
node dev/demo-marauder-ops.ts     # LIVE ops walk: scan + sniff + stop + list + save
node dev/test-serial.ts      # full live-device harness (Flipper must be plugged in)
npx tsc --noEmit             # typecheck
PI_FLIPPER_DEBUG=1 …         # timestamped chunk logs to stderr
```

## Adding a device (protocol-pluggable core)

The engine is device-agnostic: device specifics live in a
`ProtocolProfile` (`extension/profiles.ts`). To support a new device,
describe its prompt pattern, line terminator, abort byte (if any),
baud, response-echo shape and port discovery — no engine changes.
Marauder is the second profile on the same core
(`new SerialSession(port, marauderProfile)`); the bridge client tools and
direct-USB / AIO board status are live (tickets #9–#10).
