# Phase 1: ZMQ Simulation Lab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a fully software, no-RF-hardware 5G Standalone network (Open5GS core + srsRAN Project gNB + srsRAN_4G UE, wired together over ZeroMQ instead of real radio) in Docker Compose, provision a test subscriber, and prove basic registration/attach end-to-end via a single repeatable smoke test.

**Architecture:** Vendor the community-maintained `herlesupreeth/docker_open5gs` reference repo as a git submodule (it already ships known-good Open5GS 5GC + srsRAN ZMQ compose fragments) rather than hand-authoring ~12 interdependent Open5GS network-function configs from scratch. Our own code is a thin `lab/` layer on top: environment config, a subscriber-provisioning script against Open5GS's REST API, and a smoke-test script that brings the stack up in the correct order and asserts success from container logs.

**Tech Stack:** Docker Compose, Open5GS (5GC), srsRAN Project (gNB, ZMQ driver), srsRAN_4G (srsue, ZMQ driver), bash, curl, openssl.

## Global Constraints

- This plan implements **Phase 1 (ZMQ simulation) only**, per `docs/superpowers/specs/2026-08-08-private-5g-test-network-design.md`. Phase 2 (coax-connected bladeRF) and Phase 3 (OTA, regulatorily gated) are separate follow-on plans — no bladeRF hardware is touched by any task here.
- Core network: Open5GS 5GC, vendored via `herlesupreeth/docker_open5gs` (git submodule), not hand-written.
- Test PLMN: MCC `001` / MNC `01` (3GPP reserved test value) — never a real operator's PLMN.
- Host: native Linux, bare metal, Docker + Docker Compose v2 (`docker compose`, not the standalone `docker-compose` v1 binary).
- No physical SIM, no real modem, no bladeRF in this plan — srsUE reads credentials from its config file.

---

### Task 1: Vendor the Open5GS/srsRAN reference stack

**Files:**
- Create: `.gitmodules`
- Create (via submodule): `vendor/docker_open5gs/` (entire vendored repo tree)

**Interfaces:**
- Produces: `vendor/docker_open5gs/sa-deploy.yaml` (Open5GS 5GC compose fragment), `vendor/docker_open5gs/srsgnb_zmq.yaml` (gNB compose fragment), `vendor/docker_open5gs/srsue_5g_zmq.yaml` (UE compose fragment) — later tasks reference these three files by these exact paths.

- [ ] **Step 1: Add the submodule**

```bash
git submodule add https://github.com/herlesupreeth/docker_open5gs.git vendor/docker_open5gs
```

- [ ] **Step 2: Verify the expected compose fragments exist**

```bash
ls vendor/docker_open5gs/sa-deploy.yaml vendor/docker_open5gs/srsgnb_zmq.yaml vendor/docker_open5gs/srsue_5g_zmq.yaml
```

Expected: all three paths print with no "No such file" error. If any filename differs (upstream repo layout may have shifted since this plan was written), run `ls vendor/docker_open5gs/*.yaml` and `ls vendor/docker_open5gs/*.yml`, identify the actual filenames for the 5G SA core deploy fragment, the srsRAN gNB ZMQ fragment, and the srsRAN 5G UE ZMQ fragment, and use those real names for every later task in this plan instead of the names given here.

- [ ] **Step 3: Locate the env template**

```bash
find vendor/docker_open5gs -maxdepth 1 -iname '*.env*'
```

Expected: one file (commonly `.env` or `.env.example`/`.env.template`). Note its exact name for Task 2.

- [ ] **Step 4: Commit**

```bash
git add .gitmodules vendor/docker_open5gs
git commit -m "chore: vendor docker_open5gs as submodule for Open5GS/srsRAN ZMQ stack"
```

---

### Task 2: Lab environment configuration

**Files:**
- Create: `lab/.env`
- Create: `lab/README.md`

**Interfaces:**
- Consumes: the env template file found in Task 1 Step 3.
- Produces: `lab/.env` with variables `MCC`, `MNC`, `TAC`, `UE1_IMSI`, `UE1_KI`, `UE1_OP` (or `UE1_OPC` — match whichever key the vendored template actually uses), and whatever static-IP/subnet variables the vendored template defines. Later tasks pass `--env-file ../../lab/.env` (path relative to `vendor/docker_open5gs/`) to every `docker compose` invocation.

- [ ] **Step 1: Copy the vendored env template**

```bash
cp vendor/docker_open5gs/<env-template-name-from-task-1> lab/.env
```

(Replace `<env-template-name-from-task-1>` with the real filename found in Task 1 Step 3.)

- [ ] **Step 2: Confirm test PLMN values**

```bash
grep -E '^(MCC|MNC|TAC)=' lab/.env
```

Expected: `MCC=001` and `MNC=01` (the 3GPP reserved test PLMN — do not change these to a real operator's values). If the template has different defaults, edit `lab/.env` so these three lines read exactly `MCC=001`, `MNC=01`, `TAC=1`.

- [ ] **Step 3: Confirm test subscriber defaults are present**

```bash
grep -E '^UE1_' lab/.env
```

Expected: lines for the UE's IMSI, K (`UE1_KI` or `UE1_K`), and OPc (`UE1_OP` or `UE1_OPC`). Leave the vendored defaults as-is for this task — Task 4 provisions this exact subscriber into Open5GS's database, so the IMSI/K/OPc used there must match whatever is in this file.

- [ ] **Step 4: Write `lab/README.md`**

```markdown
# Phase 1 — ZMQ Simulation Lab

No RF hardware required. Brings up Open5GS 5GC + srsRAN gNB + srsRAN UE
wired over ZeroMQ instead of real radio.

Test PLMN: 001/01 (3GPP reserved test value). Test subscriber IMSI/K/OPc
are defined in `lab/.env` and provisioned into Open5GS by
`lab/scripts/provision_subscriber.sh`.

Run the whole thing with `lab/scripts/smoke_test.sh`.
```

- [ ] **Step 5: Commit**

```bash
git add lab/.env lab/README.md
git commit -m "feat: add lab environment config for ZMQ simulation stack"
```

---

### Task 3: Bring up the Open5GS core and verify it's healthy

**Files:**
- Create: `lab/scripts/core_up.sh`
- Create: `lab/scripts/core_down.sh`

**Interfaces:**
- Consumes: `lab/.env`, `vendor/docker_open5gs/sa-deploy.yaml` (or the real filename from Task 1).
- Produces: a running Open5GS core reachable at the WebUI port defined in `lab/.env` (commonly `9999`). Task 4 depends on this being up.

- [ ] **Step 1: Write `lab/scripts/core_up.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml --env-file ../../lab/.env up -d
```

```bash
chmod +x lab/scripts/core_up.sh
```

- [ ] **Step 2: Write `lab/scripts/core_down.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml --env-file ../../lab/.env down
```

```bash
chmod +x lab/scripts/core_down.sh
```

- [ ] **Step 3: Run it and verify all containers are healthy**

```bash
./lab/scripts/core_up.sh
sleep 15
docker compose -f vendor/docker_open5gs/sa-deploy.yaml --env-file lab/.env ps
```

Expected: every service (`mongo`, `webui`, `nrf`, `scp`, `ausf`, `udr`, `udm`, `smf`, `upf`, `amf`, `pcf`, `bsf`, `nssf`) shows `Up` (or `running`/`healthy` depending on Compose version's status label) — none in `Restarting` or `Exited`.

- [ ] **Step 4: Verify the AMF is alive**

```bash
docker compose -f vendor/docker_open5gs/sa-deploy.yaml --env-file lab/.env logs amf | tail -30
```

Expected: no repeated crash/restart loop in the log tail; AMF process is up and waiting for NG connections (it will not have any gNB connected yet — that's expected at this point).

- [ ] **Step 5: Verify the WebUI is reachable**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9999/
```

Expected: `200` (adjust the port if `lab/.env` defines a different WebUI port than 9999 — check with `grep -i webui lab/.env` if this fails).

- [ ] **Step 6: Tear down and commit**

```bash
./lab/scripts/core_down.sh
git add lab/scripts/core_up.sh lab/scripts/core_down.sh
git commit -m "feat: add scripts to bring Open5GS core up/down"
```

---

### Task 4: Provision the test subscriber via Open5GS's REST API

**Files:**
- Create: `lab/scripts/provision_subscriber.sh`

**Interfaces:**
- Consumes: `lab/.env` (`UE1_IMSI`, `UE1_KI`/`UE1_K`, `UE1_OP`/`UE1_OPC`), the running WebUI from Task 3 (`http://127.0.0.1:9999`).
- Produces: a subscriber record in Open5GS's MongoDB matching the UE's config — Task 6 (UE bring-up) depends on this existing before the UE attempts to attach, or it will fail auth with "Cannot find IMSI in DB".

- [ ] **Step 1: Write `lab/scripts/provision_subscriber.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../.."
set -a
source lab/.env
set +a

WEBUI_URL="${WEBUI_URL:-http://127.0.0.1:9999}"

curl -sf -X POST "${WEBUI_URL}/api/db/subscriber" \
  -H 'Content-Type: application/json' \
  -d @- <<JSON
{
  "imsi": "${UE1_IMSI}",
  "security": {
    "k": "${UE1_KI:-${UE1_K:-}}",
    "op": null,
    "opc": "${UE1_OP:-${UE1_OPC:-}}",
    "amf": "8000",
    "sqn": 0
  },
  "slice": [{
    "sst": 1,
    "default_indicator": true,
    "session": [{
      "name": "internet",
      "type": 3,
      "ambr": {
        "uplink": {"value": 1, "unit": 3},
        "downlink": {"value": 1, "unit": 3}
      },
      "qos": {
        "index": 9,
        "arp": {
          "priority_level": 8,
          "pre_emption_capability": 1,
          "pre_emption_vulnerability": 1
        }
      }
    }]
  }]
}
JSON
echo
```

```bash
chmod +x lab/scripts/provision_subscriber.sh
```

- [ ] **Step 2: Bring the core up and run it**

```bash
./lab/scripts/core_up.sh
sleep 15
./lab/scripts/provision_subscriber.sh
```

Expected: the `curl -f` call does not error out (curl's `-f` makes it exit non-zero on HTTP 4xx/5xx). If it fails with a 4xx, inspect the response by re-running the `curl` command from the script without `-f` and `-s` to see the body — the most likely cause is that the request body's field names don't exactly match this Open5GS version's schema, in which case check the live WebUI's own "add subscriber" form/network request (via browser dev tools against `http://127.0.0.1:9999`) for the exact shape and adjust the script's JSON to match.

- [ ] **Step 3: Verify the subscriber exists**

```bash
source lab/.env
curl -s "http://127.0.0.1:9999/api/db/subscriber/${UE1_IMSI}" | grep -q "${UE1_IMSI}" && echo FOUND
```

Expected: prints `FOUND`.

- [ ] **Step 4: Tear down and commit**

```bash
./lab/scripts/core_down.sh
git add lab/scripts/provision_subscriber.sh
git commit -m "feat: add subscriber provisioning script against Open5GS REST API"
```

---

### Task 5: Bring up the gNB (ZMQ) and verify NG setup with the core

**Files:**
- Create: `lab/scripts/gnb_up.sh`
- Create: `lab/scripts/gnb_down.sh`

**Interfaces:**
- Consumes: `lab/.env`, `vendor/docker_open5gs/srsgnb_zmq.yaml`, a running core (Task 3).
- Produces: a running gNB with an established N2 (NGAP) connection to the AMF. Task 6 (UE) depends on this gNB being up and connected before it can attach.

- [ ] **Step 1: Write `lab/scripts/gnb_up.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml -f srsgnb_zmq.yaml --env-file ../../lab/.env up -d
```

```bash
chmod +x lab/scripts/gnb_up.sh
```

- [ ] **Step 2: Write `lab/scripts/gnb_down.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml -f srsgnb_zmq.yaml --env-file ../../lab/.env down
```

```bash
chmod +x lab/scripts/gnb_down.sh
```

- [ ] **Step 3: Run it and check the gNB's own logs for NG setup success**

```bash
./lab/scripts/gnb_up.sh
sleep 20
docker compose -f vendor/docker_open5gs/sa-deploy.yaml -f vendor/docker_open5gs/srsgnb_zmq.yaml --env-file lab/.env logs gnb | grep -iE 'ng setup|n2 accepted|amf connected' || true
```

Expected: at least one line indicating the NG/N2 interface to the AMF came up successfully. If nothing matches, dump the full gNB log (`docker compose ... logs gnb`) and check for PLMN/TAC mismatch between `srsgnb_zmq.yaml`'s templated `cell_cfg`/`cu_cp.amf` values and `lab/.env`'s `MCC`/`MNC`/`TAC` — this is the most common cause of NG setup failure in this stack.

- [ ] **Step 4: Tear down and commit**

```bash
./lab/scripts/gnb_down.sh
./lab/scripts/core_down.sh
git add lab/scripts/gnb_up.sh lab/scripts/gnb_down.sh
git commit -m "feat: add scripts to bring gNB (ZMQ) up/down"
```

---

### Task 6: Bring up the UE (ZMQ) and verify registration + PDU session

**Files:**
- Create: `lab/scripts/ue_up.sh`
- Create: `lab/scripts/ue_down.sh`

**Interfaces:**
- Consumes: `lab/.env`, `vendor/docker_open5gs/srsue_5g_zmq.yaml`, a running core+gNB (Tasks 3, 5), a provisioned subscriber (Task 4).
- Produces: the end-to-end success signal this whole plan is building toward — the UE's own log lines `RRC Connected` and `PDU Session Establishment successful`. Task 7's smoke test asserts on these exact strings.

- [ ] **Step 1: Write `lab/scripts/ue_up.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml -f srsgnb_zmq.yaml -f srsue_5g_zmq.yaml --env-file ../../lab/.env up -d
```

```bash
chmod +x lab/scripts/ue_up.sh
```

- [ ] **Step 2: Write `lab/scripts/ue_down.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../../vendor/docker_open5gs"
docker compose -f sa-deploy.yaml -f srsgnb_zmq.yaml -f srsue_5g_zmq.yaml --env-file ../../lab/.env down
```

```bash
chmod +x lab/scripts/ue_down.sh
```

- [ ] **Step 3: Bring up the full stack in order and provision the subscriber**

```bash
./lab/scripts/core_up.sh
sleep 15
./lab/scripts/provision_subscriber.sh
./lab/scripts/gnb_up.sh
sleep 20
./lab/scripts/ue_up.sh
sleep 20
```

- [ ] **Step 4: Check the UE's logs for the two success lines**

```bash
docker compose -f vendor/docker_open5gs/sa-deploy.yaml -f vendor/docker_open5gs/srsgnb_zmq.yaml -f vendor/docker_open5gs/srsue_5g_zmq.yaml --env-file lab/.env logs ue | grep -E 'RRC Connected|PDU Session Establishment successful'
```

Expected: both lines appear. If `RRC Connected` is missing, check the UE's `[usim]` IMSI/K/OPc in `srsue_5g_zmq.yaml`'s templated values against `lab/.env`'s `UE1_*` variables and against what Task 4 actually provisioned (Task 4 Step 3's `GET` should show the same IMSI). If `RRC Connected` appears but `PDU Session Establishment successful` doesn't, check the network namespace exists (`ip netns list` should show a UE namespace) — this is set up by the vendored `srsue_5g_zmq.yaml` container entrypoint, not by our scripts.

- [ ] **Step 5: Tear everything down and commit**

```bash
./lab/scripts/ue_down.sh
git add lab/scripts/ue_up.sh lab/scripts/ue_down.sh
git commit -m "feat: add scripts to bring UE (ZMQ) up/down, verified attach end-to-end"
```

---

### Task 7: End-to-end smoke test script

**Files:**
- Create: `lab/scripts/smoke_test.sh`
- Modify: `README.md` (repo root — create if it doesn't exist yet)

**Interfaces:**
- Consumes: every script from Tasks 3–6 (`core_up.sh`, `provision_subscriber.sh`, `gnb_up.sh`, `ue_up.sh`, and the corresponding `_down.sh` scripts).
- Produces: a single command, `lab/scripts/smoke_test.sh`, that is the plan's overall acceptance test — this is what "Phase 1 works" means going forward, and what Phase 2's plan will re-run unmodified against real hardware as its own regression check.

- [ ] **Step 1: Write `lab/scripts/smoke_test.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../.."
VENDOR=vendor/docker_open5gs

cleanup() {
  echo "--- tearing down ---"
  ./lab/scripts/ue_down.sh || true
  ./lab/scripts/gnb_down.sh || true
  ./lab/scripts/core_down.sh || true
}
trap cleanup EXIT

echo "--- bringing up core ---"
./lab/scripts/core_up.sh
sleep 15

echo "--- provisioning subscriber ---"
./lab/scripts/provision_subscriber.sh

echo "--- bringing up gNB ---"
./lab/scripts/gnb_up.sh
sleep 20

if ! docker compose -f "$VENDOR/sa-deploy.yaml" -f "$VENDOR/srsgnb_zmq.yaml" --env-file lab/.env logs gnb | grep -iqE 'ng setup|n2 accepted|amf connected'; then
  echo "FAIL: gNB did not report a successful NG setup with the AMF"
  exit 1
fi

echo "--- bringing up UE ---"
./lab/scripts/ue_up.sh
sleep 20

LOGS=$(docker compose -f "$VENDOR/sa-deploy.yaml" -f "$VENDOR/srsgnb_zmq.yaml" -f "$VENDOR/srsue_5g_zmq.yaml" --env-file lab/.env logs ue)

if ! grep -q 'RRC Connected' <<<"$LOGS"; then
  echo "FAIL: UE did not report RRC Connected"
  exit 1
fi

if ! grep -q 'PDU Session Establishment successful' <<<"$LOGS"; then
  echo "FAIL: UE did not report a successful PDU session establishment"
  exit 1
fi

echo "PASS: registration and PDU session establishment succeeded end-to-end"
```

```bash
chmod +x lab/scripts/smoke_test.sh
```

This script is called as `lab/scripts/smoke_test.sh` from the repo root (matching Step 2 below) and internally `cd`s to the repo root itself, so it also works if invoked via an absolute path or symlink.

- [ ] **Step 2: Run it**

```bash
./lab/scripts/smoke_test.sh
```

Expected: final output line is `PASS: registration and PDU session establishment succeeded end-to-end`, and the script tears everything down afterward (via the `trap cleanup EXIT`) regardless of pass/fail.

- [ ] **Step 3: Write repo root `README.md`**

```markdown
# sdr-5gtool

Private, self-hosted 5G SA test network for protocol research. See
`docs/superpowers/specs/2026-08-08-private-5g-test-network-design.md` for
the full design.

## Phase 1: ZMQ simulation (no hardware required)

    git submodule update --init --recursive
    lab/scripts/smoke_test.sh

See `lab/README.md` for details.
```

- [ ] **Step 4: Commit**

```bash
git add lab/scripts/smoke_test.sh README.md
git commit -m "feat: add end-to-end smoke test for Phase 1 ZMQ simulation lab"
```

---

## What's next (not part of this plan)

Once this plan's smoke test passes reliably, the next brainstorming/design cycle covers **Phase 2**: re-running this same smoke test's assertions against the bladeRF 2.0 micro xA9 over a coax+attenuator connection instead of ZMQ, per the design spec. That requires its own plan (different RF driver config, hardware-in-the-loop steps that can't be scripted the same way) and is intentionally out of scope here.
