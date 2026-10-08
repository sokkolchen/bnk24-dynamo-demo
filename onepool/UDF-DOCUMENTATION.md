# BNK 2.4 vs NVIDIA Dynamo EPP and llm-d – AI Inference Routing Demo

Four routers take turns in front of the **same four simulated GPU workers**, with the same traffic. Only the gateway and the worker picker change. BNK 2.4 keeps the slowest users fast on a mixed GPU fleet, because it sends each request to a worker that already has its prompt cached and is not overloaded.

- **Who it is for** F5 SEs / SAs demoing AI inference delivery to network, platform and AI teams. No Kubernetes or GPU knowledge needed to run it.
- **Start-up** the lab needs **about 10 minutes** after Start before it is ready: start it before the meeting.
- **Full demo guide (PDF)** BNK 2.4 vs NVIDIA Dynamo EPP and llm-d — Demo Guide *(SharePoint link to be added)*

---

## 1. What is inside

![How each router picks a GPU worker](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/routers-onepool.png)

Four routers, one after another, in front of the **same 4 GPU workers** (llm-d inference simulator: vLLM-compatible, real KV-cache behaviour, timed like an H100 serving an 8B model):

- **BNK 2.4** — Who picks the worker: **the BNK 2.4 EPP, asked by TMM for every request** · What it knows: **cached prompt blocks + each worker's queue, running requests and KV usage + predicted time to first token per worker**
- **Istio + NVIDIA Dynamo EPP** — NVIDIA's endpoint picker (Dynamo 1.5, standalone mode) · knows cached prompt blocks + the load it booked itself
- **Istio + llm-d Optimized** — llm-d's default configuration · estimates the cache from its own past routing + in-flight tokens
- **Istio + llm-d Precise** — llm-d's precise prefix-cache routing · knows cached prompt blocks + in-flight tokens
- *Optional:* **HAProxy** round robin as a baseline (adds ~7 minutes to a run).

Every picker runs its **vendor-documented configuration**. All KV-aware pickers receive the same KV-cache events from the workers (ZMQ, port 20080).

![Lab network](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/network-onepool.png)

- The client reaches the GPU cluster through a top-of-rack router with BGP and two equal-cost paths.
- Every request enters through an **emulated DPU** (VM + Open vSwitch, wired like F5's BlueField-3 design). **BNK's TMM runs on the DPU.**
- All routers use the same network path, so the comparison is fair.

**Be honest with customers:** the workers are simulated and the DPU is emulated, so the numbers are lab numbers. The routing software, KV caches and KV events are real.

---

## 2. How to start (no manual steps)

1. **Deploy** this Blueprint (or **Start** your existing deployment).
2. **Wait about 10 minutes.** The lab prepares itself on every start:
   - Kubernetes, BNK, the 4 GPU workers and the 4 worker pickers come up;
   - a start-up script points the dashboard buttons at *this* deployment;
   - it checks every router with a test request.
3. Open **node1 → Access → GRAFANA**. No login is needed to view and run the demo.
4. Watch the **Lab** tile, top right of the dashboard:

- **🟡 *Lab warming up (~10 min after start)…*** — start-up still running: wait (demo buttons are refused until it is done)
- **🔵 *Lab ready – GPUs idling*** — ready: run a scenario
- **🟠 *Cooking tokens…*** — a demo run is in progress
- **🟢 *Served hot*** — last run finished, results on screen
- **🔴 *Burnt*** — problem: see section 6

> ⚠️ **We recommend not using Force Stop** on this deployment. In our tests a Force Stop wiped the node disks, and the lab then needed a full rebuild. The normal **Stop** is safe.

---

## 3. How to run the demo

![Dashboard: scenario buttons, Routers selector and Lab tile](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/dashboard-top.png)

Grafana opens on **★ MAIN** (folder *BNK 2.4 AI Demo*). Click a scenario button; a small tab opens and closes, and the run starts. The routers are tested one after another, each from empty KV caches; results appear per router. The **Routers** selector chooses who runs: **Default 4** (BNK 2.4, Dynamo EPP, llm-d Optimized, llm-d Precise) or **Default 4 + HAProxy**.

| Scenario | What it shows | Pool / prompt / cache / load | Duration |
|---|---|---|---|
| **S4 — Identical GPUs, many short-prompt apps** | best cache hit and shortest tail on equal GPUs: **a good first live run** | equal / 2k / 195 % / 100 % | ~6 min |
| **S1 — Mixed GPU fleet at full capacity, many apps** | BNK keeps the slow GPU's queue short and the slowest users fast | mixed / 4k / 130 % / 100 % | ~10 min |
| **S2 — Long RAG documents, heavy cache pressure** | exact cache placement + speed awareness | mixed / 8k / 195 % / 90 % | ~10 min |
| **S3 — Long prompts, busy hour** | vs llm-d's default configuration | mixed / 8k / 65 % / 90 % | ~10 min |

Cache % = all different shared prompts as a share of ONE GPU's KV cache. Load % = share of the pool's measured capacity.

![Results after scenario S1](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/S1-headline.png)

![Which GPU worker got the requests (scenario S1)](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/S1-workers.png)

**What the results show** (full test grid: 72 operating points, 3 runs per point and router, medians):

![On mixed GPU fleets, BNK 2.4 keeps the slowest users fast at every tested point](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/results-1-summary.png)

![As load rises, other pickers break more often; BNK 2.4 never does](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/results-2-trend.png)

![The BNK 2.4 message: slowest users stay fast on mixed GPU fleets](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/onepool/img/results-3-message.png)

Custom runs are possible: set *Pool*, *Prompt tokens*, *Cache level*, *Load* and *Answer tokens* in the bar at the top, then click **Run with my settings**. Limits are enforced.

**Other dashboards:** *BACKUP (classic)* = the same results in a compact layout. *BNK 2.4 — production traffic (live)* = endless random traffic to BNK 2.4 only (Start / Stop buttons), to show live behaviour.

---

## 4. Access and credentials

- **Grafana (demo)**: node1 → Access → **GRAFANA**. Login: none needed to view/run. Admin: `admin` / `Junct10n!` (only to edit dashboards)
- **Demo API (called by the buttons)**: jumphost → Access → **DEMO API**. Login: none
- **Prometheus**: node1 → Access → **PROMETHEUS**. Login: none
- **Jumphost shell**: jumphost → Access → **Web Shell** or SSH. Login: your UDF key

---

## 5. What is on the jumphost (ready for Claude Code)

All lab automation lives on the jumphost and survives restarts. Two entry files explain it:

- `~/README.md`: for people. Covers the start-up sequence, the file map, the topology and the known pitfalls.
- `~/CLAUDE.md`: the same context for an AI coding assistant. Open **Claude Code** (or another assistant) in the jumphost home directory and it picks this file up.

Structure (short):

```
~/README.md, ~/CLAUDE.md     start here
~/lab-boot.log               start-up log ("LAB READY")
~/lab/
  lab-boot.sh (+ .service)   start-up automation, runs on every start
  path-check.sh              health check (routes, ECMP, chat per router, workers + pickers, BNK, licence, monitoring)
  pool-ready.sh              worker pool check (4 workers, KV-event subscriptions, HAProxy)
  demo-api.py, demo/         Grafana buttons -> scenarios -> selected routers one after another -> results
  grafana-dash.sh            regenerate + upload the dashboards
  routers.json               the router addresses (single source)
  prodtraffic.py             production-traffic dashboard (BNK 2.4 only)
  relicense.sh               BNK 30-day eval licence, renewed at start
  inline/, REVIVE.md         in-line networking, what must survive a restart
~/llmd/scripts/llmd-pool.sh  worker pool: mixed-speed / equal / reset
~/onepool/                   how the one-pool stack was built (manifests, configs, test scripts)
```

---

## 6. If something looks wrong (jumphost → Web Shell)

- **Lab tile yellow for more than 20 min** → `tail ~/lab-boot.log` shows the step it waits for
- **Lab tile red (Burnt) right after start** → `bash ~/lab/lab-boot.sh` (~1 min once pods are up)
- **A button does nothing / "busy"** → `curl -s localhost:5050/status` on the jumphost; failed steps are logged in `~/lab/demo-api.log`
- **Before an important demo** → `bash ~/lab/path-check.sh` → last line must be `PATH CHECK: OK`
- **Still broken** → normal **Restart** of the deployment (we recommend not using Force Stop), wait 10–15 min

---

*Lab facts:*
- UDF, 9 VMs.
- Kubernetes 1.35.
- BNK 2.4.0. It uses a 30-day evaluation licence, which renews automatically at start.
- NVIDIA Dynamo 1.5 EPP (standalone mode), llm-d 0.11 router, Istio 1.29 with the Gateway API Inference Extension.
- 4 llm-d-inference-sim workers (vLLM-compatible): mixed pool 2 fast (2×), 1 medium (1×), 1 slow (0.3×) or 4 equal.
- MetalLB and VyOS routers.
