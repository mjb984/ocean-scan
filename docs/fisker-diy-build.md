# Fisker Ocean — DIY Communications Build & Pulse Replacement

Owner: Mauricio · Car: Fisker Ocean, firmware 2.2.3 · Last updated: 2026-11-26
Principle: fully self-hosted. No OV Loop, no FOA subscriptions, no vendor cloud. Everything reports to Mauricio's own server.

## Why this project exists

The OceanLink Pulse (EV Play-based Android/CAN box, ReTechImagine/Undercurrent, lead dev Majd Srour) has been on pre-order for over a year. Mauricio's order is **Batch 4**.

Pulse production status (read from the FiskerOcean Discord, 2026-11-26):
- The Orange Pi 5 Pro the Pulse was built on became unavailable in the global memory/storage chip shortage. Production switched to the **Radxa Rock 5C**.
- 400 Rock 5C boards covering EU Batch 1 + NA Batch 2 were still in transit as of Nov 13 (per Majd Srour); no arrival confirmed by Nov 25. All batches are delayed on hardware supply.
- Almost everyone paid in full has been fulfilled; unpaid orders have no board allocated. After this batch, the batch system is dropped — pay in full, board gets ordered, shipping follows order number.
- Batch 4 is realistically many months out. The announcements channel carries only software releases (latest Nov 25, 2026: build 2864, app v1.0.1).

Decision: build our own equivalent instead of waiting.

## Architecture — two boxes, two jobs

The Pulse is a commodity RK3588-class Android SBC + a proprietary display-intercept harness + vendor software/cloud. We replicate the hardware concept, replace the cloud with Mauricio's server, and split the job:

1. **Comms/diagnostics box — Raspberry Pi 5 (4GB) on Linux.** CAN listening/logging, UDS diagnostics, vehicle status relayed to Mauricio's own server (Tailscale/MQTT). This replaces the Pulse's paid cloud half with something better: ours.
2. **Infotainment box — Radxa Rock 5C (RK3588S2, 8GB) on Android.** Wireless CarPlay/Android Auto receiver, native Android apps, HDMI out. Same board family the Pulse itself now uses. Bench-proven on a normal monitor first; drives the car's center display only after the display intercept (Phase 3) is cracked.

## Phase 1 — Comms/logging (parts researched, verified pricing 2026-11-26)

- Raspberry Pi 5 4GB — $110, PiShop.us
- Waveshare 2-Channel Isolated CAN FD Expansion HAT — $46.99 direct from Waveshare (MCP2518FD, 2 channels — the Ocean has several buses)
- Kaizen Speed OBD2 pass-through harness — $25 (no-cut tap for CAN / 12V / ground)
- Geekworm Pi 5 5V 5A dual PD power module — $16.50 (9–24V in; pick the 3.81mm terminal-block variant)
- Optional: Geekworm supercapacitor UPS HAT (60F) — $46.90, bridges power loss for clean shutdown
- All-in ≈ $200 with the Pi, vs Pulse $399 + subscription
- CarPiHAT PRO 5 (the all-in-one alternative: 12V→5V 5A, safe shutdown, CAN, RTC, DAC) is £116.80 at The Pi Hut but out of stock; a daily restock watch is running. Note its single CAN channel vs the Waveshare's two.

### Parked/sleep design rules (from owner-community tools: Ocean Live, Unfiskered)

- The Ocean deep-sleeps minutes after locking with the fob out of range; buses go quiet.
- Never poll to keep the car awake — it drains the 12V battery. Owner tools back off and never send keep-alives.
- Design is event-driven: Pi fully off when parked (<1mA class draw), woken by ignition/CAN activity (unlock, charge start) or its own modem; brief poll, report, shut back down.
- Parked remote status = last-known state + wake-event updates. A sleeping OBD adapter draws ~3mA and can read 12V voltage without waking the car.
- Pi 5 idles ~2.5–3.5W — always-on would drain the 12V in about a week. Ignition-switched power + safe shutdown is mandatory.
- Verify the Ocean's OBD pinout before connecting anything (standard J1962: 6=CAN-H, 14=CAN-L, 16=+12V, 4/5=ground — not yet verified on this car).

## Phase 2 — UDS diagnostics

- The Ocean is **UDS-only**: generic OBD broadcast queries return nothing. The gateway answers per-ECU UDS requests on each ECU's CAN ID (BMS, BCM, ECC, ICC, iBooster, ~39 modules mapped by community tools), including NRC 0x78 (response pending) handling.
- Reads mostly need no security access. Clears/routines need per-ECU seed-key — later work.
- Linux speaks ISO-TP natively (can-isotp), so the Pi is a better host than a BLE dongle. Start read-only.

## Phase 3 — Display intercept (the research project)

EV Play/Pulse sits **between** the ICC (infotainment computer) and the center display: taps video + CAN, switches sources, auto-reverts for the reverse camera, touch passes through. The Ocean's display connector/video format is **not publicly documented** — this is the Pulse's proprietary piece and the real work of this phase.

Plan:
1. Buy a salvage ICC + center display + interconnect harness from a wrecked Ocean; crack the link on the bench, not in the car.
2. Identify the serializer/deserializer chips on the ICC board. The Ocean's ADAS side is confirmed TI **FPD-Link III**; FPD-Link III and GMSL are the prime suspects for the display link.
3. The connectors are **FAKRA** — the standard automotive coax family. FAKRA pigtails, male/female pass-through extensions, and TI/ADI serializer/deserializer eval boards with FAKRA ports are commodity parts, so a no-cut inline tap is physically straightforward once the link protocol, resolution, and timing are known.
4. Intercept must fail safe: stock video by default, reverse camera instantly, stock UI always reachable.

### Why not replace the ICC outright?

The ICC is not just a media player — it renders vehicle settings, climate, charging and drive-mode UI, and other modules expect it on the bus. Deleting it means reimplementing an OEM head unit's vehicle integration, and any failure takes out core vehicle functions. The intercept architecture keeps the ICC doing vehicle functions and only borrows its screen: ~95% of the replacement experience at a fraction of the risk. Consumer boards (Pi/Jetson) are also wrong for this job — weak/no Android support, slow boot, not automotive-grade. The RK3588 is the right chip, which is why the Pulse uses it.

## Phase 4 — openpilot (watch and wait)

- July 2026: Majd Srour demonstrated a Comma Four lane-centering an Ocean using reverse-engineered EPS torque CAN messages. Steering only — no accel/brake. Branch not merged into public openpilot, no public release yet.
- Blockers: ADAS-bus steering/accel/brake messages are SecOC-signed (MAC + freshness); radar sits on a private CAN FD bus to the Hydra ECU in the trunk. (Majd's write-ups note a Magna SecOC implementation flaw in the radars that may ease this.)
- The Pi won't run openpilot (comma hardware + panda does), but our Phase 1/2 CAN logging and DBC work is exactly the groundwork a port needs.

## Public work we can piggyback on

- **Majd Srour's Medium series** (medium.com/@majd.srour) — multi-part: CAN sniffing and DTC decoding on the Ocean, plus openpilot integration notes (network topology, ADAS bus, SecOC). Also his May 2025 Fisker cloud security research.
- **Community CAN DBC files on GitHub** — the Ocean's buses (CCAN, PTCAN, Inverter CAN, BCAN, all 500 kbps) are being systematically mapped.
- **Ocean Live** (oceanlive.org, source on GitHub) — free owner tool reading DTCs from ~39 modules via BLE ELM327; its sleep/back-off behavior is our parked-comms reference.
- **Fiskeretta diagnostics** (GitHub) — free, community-built DTC read/clear tool.
- **MichaelOE's Home Assistant integration** — reverse-engineered the My Fisker app API; useful API reference.
- Not public: anything about the Pulse's display intercept or EV Play internals. That part we reverse engineer ourselves (Phase 3).

## Android box sourcing (checked 2026-11-26)

Official Radxa pricing at Arace Tech (arace.tech): Rock 5C 2GB $69 (in stock), 4GB $99 (out of stock), 8GB $159 (out of stock), 16GB $119.90 (out of stock), 32GB $199.90 (out of stock).
In stock today: Arace 2GB at $69; Radxa's official AliExpress store — 4GB $151.79, 8GB $235.39 (+$13.93 shipping, delivery ~Dec 4–11). ameriDroid is out of stock and priced above both. Mauricio reports Amazon has the board at $230 and will check eBay (also the likely source for the salvage ICC/display).
2GB is fine as a bench/dev unit; the car unit wants 8GB (Android + streaming + AA/CarPlay receiver).

## Open questions

- Ocean OBD pinout verification (before any connection).
- Mauricio's Pulse order: order number / paid-in-full status (no order email found in his Gmail — may be under another address). Dashboard: oceanlinkpulse.com/user/dashboard.
- Salvage ICC + display sourcing and price.
- Display link protocol identification (bench phase).
