# BNK 2.4 & NVIDIA Dynamo 1.5 Integration – Value Demo

The same simulated GPU workers, the same traffic, three different routers in front of them: and BNK 2.4 keeps the slowest users fastest, because it sends each request to a GPU worker that already has its prompt cached and is not overloaded.

- **Who it is for** F5 SEs / SAs demoing AI inference delivery to network, platform and AI teams. No Kubernetes or GPU knowledge needed to run it.
- **Start-up** the lab needs **~15 min** after Start before it is ready: start it before the meeting.
- **Full demo guide (PDF)** [BNK 2.4 AI routing demo — Lab Guide](https://f5.sharepoint.com/:b:/r/sites/EMEASystemsEngineering/Shared%20Documents/Collateral%20-%20AI/UDF%20files/BNK%202.4%20+%20Dynamo/BNK24-AI-routing-demo-guide.pdf?d=w98c43c34925e4fffbb63d2b738b9990f&csf=1&web=1&e=AU0YSW) 
- **Customer Facing Demo video** [BNK 2.4 & NVIDIA Dynamo 1.5 — demo on YouTube](https://www.youtube.com/watch?v=86SrYlpJ8cI) 


---

## 1. What is inside

![How each router picks a GPU worker](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/routers.png)

Three routers take turns in front of the **same 4 GPU workers** (NVIDIA Dynamo 1.5, simulated: timed like an H100 serving an 8B model):

- **HAProxy** — Who picks the worker: round robin · What it knows: nothing about the GPUs (baseline)
- **Istio + NVIDIA Dynamo EPP** — Who picks the worker: NVIDIA's endpoint picker (Kubernetes reference design) · What it knows: cached prompt blocks + load
- **BNK 2.4** — Who picks the worker: **BNK 2.4, for every request** · What it knows: **cached prompt blocks + queue, running requests, KV usage, predicted TTFT**


![Lab network](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/network.png)

- The client reaches the GPU cluster through a top-of-rack router with BGP and two equal-cost paths.
- Every request enters through an **emulated DPU** (VM + Open vSwitch, wired like F5's BlueField-3 design). **BNK's TMM runs on the DPU.**
- All routers use the same network path, so the comparison is fair.

The lab also contains NVIDIA's **Dynamo router** (built into the Dynamo frontend). It is not part of the standard demo, but you can add it to a run with the **Routers** selector at the top of the dashboard.

**Be honest with customers:** the workers are simulated (NVIDIA's Dynamo *mocker*), and the DPU is emulated, so the numbers are lab numbers. The routing software, KV caches and KV events are real.

---

## 2. How to start (no manual steps)

1. **Deploy** this Blueprint (or **Start** your existing deployment).
2. **Wait ~10–15 minutes.** The lab prepares itself on every start:
   - Kubernetes, BNK and the GPU workers come up;
   - a start-up script points the dashboard buttons at *this* deployment;
   - it sends a test request through each router.
3. Open **node1 → Access → GRAFANA**. No login is needed to view and run the demo.
4. Watch the **Lab** tile, top right of the dashboard:

- **🟡 *Lab warming up (~10 min after start)…*** — start-up still running: wait
- **🔵 *Lab ready – GPUs idling*** — ready: run a preset
- **🟠 *Cooking tokens…*** — a demo run is in progress
- **🟢 *Served hot*** — last run finished, results on screen
- **🔴 *Burnt*** — problem: see section 6


**Expected timing after Start:**

- **+3 min** — VMs booted
- **+8–9 min** — all Kubernetes, BNK and Dynamo pods ready
- **+10 min** — start-up script done: **Lab ready**
- **up to +13 min** — UDF SSH / Web Shell access may lag behind; Grafana usually works earlier


> ⚠️ **We recommend not using Force Stop** on this deployment. In our tests a Force Stop wiped the node disks, and the lab then needed a full rebuild. The normal **Stop** is safe.

---

## 3. How to run the demo

![Dashboard: buttons and Lab tile](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/dashboard-top.png)

Grafana opens on **★ MAIN — AI Inference Routing — BNK 2.4 demo** (folder *BNK 2.4 AI Demo*). Click a preset button; a small tab opens and closes, and the run starts. The routers are tested one after another; results appear per router as each phase ends. Use the **Routers** selector at the top to choose who runs: *Standard - HAProxy + Istio + BNK* (default) or *Istio vs BNK - fastest* (~3 min for preset D). Routers that are not in a run show no bars.

- **A — Mixed GPU fleet** (2 fast, 1 medium, 1 slow worker), ~8 min: **pre-run it before the meeting**. Shows how BNK keeps the slow GPU's queue short.
- **B — Same fleet, busy hour** (75 % load), ~8 min.
- **C — Identical GPUs, long 8k-token RAG context**, ~5 min. Shows the cache hit difference.
- **D — Identical GPUs, peak hour** (100 % load), ~4 min: **a good live run**.


What the dashboard shows after a run: TTFT (time to first token) p50 / p95 / p99, end-to-end latency, throughput, prefix cache hit, errors, and which worker type got the requests, one bar per router.

![TTFT after preset A](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/ttft-A.png)

![Which worker got the requests (preset A)](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/worker-split-A.png)

**What to say about the size of the gain:**

- **Against round robin, the gain is large.** Any KV-aware router, NVIDIA's included, gets most of it.
- **Against NVIDIA's own routers, the gain is real but smaller.** It shows mostly for the slowest users (p95/p99) and in cache hit.
- **For the typical user (p50), BNK is about even with NVIDIA's routers**, and can be a little slower in some presets: the emulated DPU adds a GRE hop that a real BlueField would not.

Custom runs are possible: set *Pool*, *Prompt tokens*, *Cache level*, *Load* and *Answer tokens* in the bar at the top, then click **Run with my settings**. Limits are enforced.

**Other dashboard:** *BACKUP (classic)* is the original 3-router PoC layout. Keep it as a fallback; demo from **★ MAIN**.

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
- `~/CLAUDE.md`: the same context for an AI coding assistant. Open **Claude Code** (or another assistant) in the jumphost home directory and it picks this file up. It lists how to check and operate the lab, the hard rules, and a table of known gotchas (symptom → cause → fix).

Structure (short):

```
~/README.md, ~/CLAUDE.md     start here
~/lab-boot.log               start-up log ("LAB READY")
~/lab/
  lab-boot.sh (+ .service)   start-up automation, runs on every start
  path-check.sh              health check (routes, ECMP, chat per router, TMM, KV-event feed, licence)
  demo-api.py, demo/         Grafana buttons -> presets -> selected routers one after another -> results
  grafana-dash.sh            regenerate + upload the dashboards
  routers.json               the router addresses
  lab-profile.sh             worker pool shape (mixed-speed / equal)
  haproxy-gen.sh             HAProxy config from live workers
  bnk-ai-gw.yaml, f5-epp-*   BNK 2.4 gateway + worker-selection config
  istio-*.yaml               Istio + Dynamo EPP path
  inline/                    in-line networking: MetalLB, router BGP, DPU OVS, GRO-off
  REVIVE.md                  what must survive a restart, and why
  rebuild-full.sh            full rebuild on blank VMs (~2 h, unattended)
  archive/                   investigation scripts from building the lab
~/udf-cne, ~/run-*.sh        BNK installer with lab patches
```

---

## 6. If something looks wrong (jumphost → Web Shell)

- **Lab tile yellow for more than 20 min** → `tail ~/lab-boot.log` shows the step it waits for
- **Lab tile red (Burnt) right after start** → `bash ~/lab/lab-boot.sh` (~30 s once pods are up)
- **Before an important demo** → `bash ~/lab/path-check.sh` → last line must be `PATH CHECK: OK`
- **A router shows errors in a run** → `bash ~/lab/path-check.sh`; the red line says what to run
- **Still broken** → normal **Restart** of the deployment (we recommend not using Force Stop), wait 15 min


---

*Lab facts:*
- UDF, 9 VMs.
- Kubernetes 1.35.
- BNK 2.4.0. It uses a 30-day evaluation licence, which renews automatically at start.
- NVIDIA Dynamo 1.5 mocker workers.
- Istio 1.29 with the Gateway API Inference Extension.
- MetalLB and VyOS routers.

