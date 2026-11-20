# Private Self-Hosted 5G/LTE Test Network — Design

## Purpose

A private, self-hosted 5G (SA) test network — core + RAN + UE — run entirely by
one operator (the researcher) against their own subscribers, for hands-on
protocol research: RRC connection setup, paging, authentication/AKA, and
related layers beyond broadcast. No connection to, interception of, or
interference with any commercial carrier network.

Hardware on hand: bladeRF 2.0 micro xA9 (one unit). Host: native Linux
(bare metal), chosen specifically because Docker + bladeRF USB passthrough
and OCUDU's real-time RF path are most reliable there (VMs add USB
passthrough and scheduling jitter that hurts real-time RF timing).

## Scope & Phasing

This is one design covering three phases, sharing one Docker Compose
topology switched by a config flag:

1. **ZMQ simulation** — core + gNB + UE all in software, no RF hardware,
   zero regulatory surface. Fastest iteration for protocol logic.
2. **Coax-connected real-RF validation** — same stack, gNB and UE driven by
   the bladeRF over a direct coax + attenuator connection (no antennas, no
   radiation). Validates the actual SDR TX/RX chain, gain, and timing
   without touching spectrum regulation.
3. **Over-the-air (OTA)** — antennas attached, actual radiation. Gated on
   regulatory status (see below); not part of the initial build-out.

Each phase reuses the same core network and subscriber database; only the
RAN/UE transport changes.

## Component Choices

### Core network: Open5GS

Open5GS is the pick, for the 5G-SA-first path:
- Single codebase covers both 5GC and EPC, so a later LTE detour doesn't
  require a different core.
- MongoDB-backed subscriber store with a WebUI and REST API — this is the
  practical provisioning path for test IMSIs/keys (see below).
- Actively maintained, large community, this is also what the srsRAN Project's
  (now OCUDU's) 5G SA documentation pairs against as the reference core.
  OCUDU does not currently ship a competing core for 5G SA (srsEPC exists for legacy
  4G-only setups but isn't a live alternative once you're on the 5G SA
  path chosen here), so this isn't an either/or decision for this project.

### RAN: OCUDU gNB

Note: the srsRAN Project repository was archived on 2026-06-01; its successor
is OCUDU (gitlab.com/ocudu/ocudu, docs: docs.ocudu.org). Config and docker
references below use the `ocudu/docker` layout.

Driven by the bladeRF 2.0 micro xA9 through OCUDU's `bladerf` RF driver
(UNVERIFIED in OCUDU; pending check, tracked as a separate finding)
for real hardware (phases 2–3), or the `zmq` driver for simulation
(phase 1) — same gNB codebase, different RF backend selected by config.

### UE: srsUE (5G NR)

Software UE simulator for phases 1 and 2. A real modem + physical
programmable SIM is a deliberately deferred future swap, not designed in
now — when that happens, the changes are: (a) subscriber credentials move
from Open5GS WebUI + srsUE config file to a physically-programmed USIM,
and (b) the UE process moves from a container to real hardware attached to
the RAN over the air or via a shielded coupler. Nothing else in this
design changes.

**Hardware constraint:** with a single bladeRF, the gNB and srsUE cannot
both hold the physical device at once in phase 2/3. Either acquire a
second bladeRF (recommended if hardware-chain validation becomes a
regular activity) or run one side (typically srsUE) as a bare-metal
process against the bladeRF while gNB stays on ZMQ during any interim
step — call out explicitly in the runbook rather than silently assumed.

## Subscriber Provisioning

- Use the 3GPP-reserved test PLMN (MCC 001 / MNC 01) so test subscriber
  identifiers can never collide with a real operator's PLMN, including any
  Somali carrier's.
- Generate test IMSI / Ki (subscriber key) / OPc (operator variant key) per
  subscriber via the Open5GS WebUI (or a small reproducible script using
  `openssl` for anyone who wants deterministic test vectors under version
  control).
- srsUE reads IMSI/Ki/OPc directly from its config file — no physical SIM
  required for phases 1–2.

## RF Containment & Regulatory Status

Somalia's telecom regulator is the National Communications Authority
(NCA), operating under the Communications Act (passed 2012, signed into
law 2017). The NCA plans and manages spectrum, and its Spectrum Licensing
Regulation (2022) governs frequency assignment. Somalia's first spectrum
license went to Hormuud Telecom, covering 700MHz, 800MHz, 900MHz, 1.8GHz,
2.1GHz, 2.3GHz, 2.6GHz, and 3.5GHz — bands that overlap heavily with
common LTE/5G NR band plans. No publicly documented experimental/
test-license class (no FCC Part 15 equivalent) was found for Somalia as of
this writing.

The researcher holds a government-issued mandate to explore new-network
deployment, but as of this design it is broad/general — specific bands,
power limits, and geographic area are not yet defined in writing, and it
is not yet confirmed to be issued by or coordinated with the NCA
specifically (the body that actually controls spectrum per the above).

Given that, RF containment is phased to match actual authorization status,
not assumed authorization:

- **Phase 1 (ZMQ):** No RF hardware involved. No regulatory dependency at
  all — always safe to run regardless of licensing status.
- **Phase 2 (coax/attenuator):** Direct cable connection between bladeRF
  TX/RX ports, attenuated into the device's safe/linear range, no antennas
  attached, no radiation. This validates the real hardware signal chain
  (modulation, RF front end, timing) without engaging spectrum regulation
  at all — treated as the default and sufficient "confirm the equipment
  works" step, matching the original goal.
- **Phase 3 (OTA) — explicitly gated, not scheduled:**
  - *Current state:* not authorized to transmit over the air. The general
    mandate does not yet specify bands/power/area, and Somalia's common
    cellular bands are already licensed to at least one live commercial
    operator — transmitting there risks real interference with a
    production network.
  - *Unlock condition:* once the mandate is formalized in writing with
    specific bands, power limits, and geographic scope — ideally directly
    from or coordinated with the NCA — OTA testing becomes a concrete next
    phase. At that point, re-check the confirmed parameters against NCA's
    published Spectrum Licensing Regulation and prefer bands clear of
    Hormuud's (and any other operator's) licensed allocations to minimize
    interference risk even where technically authorized.
  - This design deliberately does not implement or schedule any OTA step;
    it only reserves the phase-3 slot in the Compose topology (see below)
    for whenever the unlock condition above is met.

## Docker Compose Topology

- One core stack (Open5GS: AMF/SMF/UPF/etc. + MongoDB + WebUI) that runs
  identically regardless of RAN/UE mode.
- RAN and UE containers are parameterized by a single env var,
  `RF_MODE=zmq|bladerf`, which selects which OCUDU config file is mounted
  in:
  - `zmq` — gNB and UE configs point at each other over `tcp://` sockets
    (ZeroMQ transport), phase 1.
  - `bladerf` — gNB and UE configs point at the physical bladeRF via USB
    passthrough, phase 2 (coax) and phase 3 (OTA) share this same mode;
    the only difference between phase 2 and 3 is whether antennas are
    physically attached, which is outside Docker's control and called out
    as a manual runbook step, not a config value.
- Compose profiles (`sim` / `hw`) gate which containers start, so
  `docker compose --profile sim up` never touches the bladeRF and
  `docker compose --profile hw up` requires the device to be present.
- Given the single-bladeRF constraint above, the `hw` profile's initial
  runbook documents running srsUE as a bare-metal process (not
  containerized) against the one physical device while gNB uses `zmq`,
  until/unless a second bladeRF is acquired.

## Research / Learning Path

1. ZMQ simulation: bring up core + gNB + UE, verify basic
   registration/attach end-to-end.
2. Still in ZMQ: exercise paging, then AKA in detail (this is where most
   of the protocol-research value is, and it's fully available in
   simulation).
3. Coax-connected real-RF validation: repeat the same registration flow
   over the physical bladeRF chain, confirm timing/gain settings hold up
   against real hardware.
4. Once 2+ cells are available (real or simulated), explore handover.
5. OTA: deferred until the phase-3 unlock condition above is met.

## Testing

- Phase 1 is the primary test environment — every protocol flow (attach,
  paging, AKA, handover once multi-cell) gets validated here first since
  it's fast to iterate and requires no hardware.
- Phase 2 is a hardware-chain smoke test, not a protocol test: same flows,
  same expected outcomes as phase 1, just confirming the bladeRF path
  doesn't introduce timing/gain issues that break what already worked in
  simulation.
- No automated CI is in scope for this lab — it's a manual research
  environment, not a shipped product.
