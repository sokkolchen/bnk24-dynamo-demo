# BNK 2.4 + Dynamo 1.5 Integration – Value Demo

**One sentence:** the same simulated GPU workers, the same traffic, four different routers in front of them: and BNK 2.4 with the F5 Endpoint Picker (F5 EPP) keeps the slowest users fastest, because it sends each request to a GPU worker that already has its prompt cached and is not overloaded.

| | |
|---|---|
| **Who it is for** | F5 SEs / SAs demoing AI inference delivery to network, platform and AI teams. No Kubernetes or GPU knowledge needed to run it. |
| **Demo length** | ~25 min (incl. one live run). Start the lab **15 min before** the meeting. |
| **Cost** | ~$2 per hour running (UDF, 64 vCPU). Stop it when you are done. |
| **Full demo guide (PDF)** | *coming soon* |
| **Demo video** | *coming soon* |
| **Owner / questions** | Alexander Serebryakov, CEE AI Solutions Architect |

---

## 1. What is inside

![How each router picks a GPU worker](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/routers.png)

Four routers take turns in front of the **same 4 GPU workers** (NVIDIA Dynamo 1.5, simulated: timed like an H100 serving an 8B model):

| Router | Who picks the worker | What it knows |
|---|---|---|
| HAProxy | round robin | nothing about the GPUs (baseline) |
| Istio + NVIDIA Dynamo EPP | NVIDIA's endpoint picker (Kubernetes reference design) | cached prompt blocks + load |
| NVIDIA Dynamo router | NVIDIA's router in the Dynamo frontend | cached prompt blocks + load |
| **BNK 2.4 + F5 EPP** | **F5 EPP, called by BNK's TMM for every request** | **cached prompt blocks + queue, running requests, KV usage, predicted TTFT** |

![Lab network](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/network.png)

- The client reaches the GPU cluster through a top-of-rack router with BGP and two equal-cost paths.
- Every request enters through an **emulated DPU** (VM + Open vSwitch, wired like F5's BlueField-3 design). **BNK's TMM runs on the DPU.**
- All four routers use the same network path, so the comparison is fair.

**Be honest with customers:** the workers are simulated (NVIDIA's Dynamo *mocker*), and the DPU is emulated, so the numbers are lab numbers. The routing software, KV caches and KV events are real.

---

## 2. How to start (no manual steps)

1. **Deploy** this Blueprint (or **Start** your existing deployment).
   Then click **EXTEND** so it does not auto-stop during your demo.
2. **Wait ~10–15 minutes.** The lab prepares itself on every start:
   - Kubernetes, BNK and the GPU workers come up;
   - a start-up script points the dashboard buttons at *this* deployment;
   - it sends a test request through each of the four routers.
3. Open **node1 → Access → GRAFANA**. No login is needed to view and run the demo.
4. Watch the **Lab** tile, top right of the dashboard:

| Lab tile | Meaning |
|---|---|
| 🟡 *Lab warming up (~10 min after start)…* | start-up still running: wait |
| 🔵 *Lab ready – GPUs idling* | ready: run a preset |
| 🟠 *Cooking tokens…* | a demo run is in progress |
| 🟢 *Served hot* | last run finished, results on screen |
| 🔴 *Burnt* | problem: see section 5 |

**Expected timing after Start** (measured on a cold Blueprint deployment, 1 Oct 2026):

| Time after Start | Status |
|---|---|
| +3 min | VMs booted |
| +8–9 min | all Kubernetes, BNK and Dynamo pods ready |
| **+10 min** | start-up script done: **Lab ready** |
| up to +13 min | UDF SSH / Web Shell access may lag behind; Grafana usually works earlier |

> ⚠️ **Never use Force Stop** on this deployment: it can lose the node disks, and the lab would then need a 2-hour rebuild. Use the normal **Stop**.

---

## 3. How to run the demo

![Dashboard: buttons and Lab tile](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/dashboard-top.png)

Grafana opens on **★ MAIN — AI Inference Routing — BNK 2.4 demo** (folder *BNK 2.4 AI Demo*). Click a preset button; a small tab opens and closes, and the run starts. The four routers are tested one after another; results appear per router as each phase ends.

| Preset | Scenario | Run time | What to show |
|---|---|---|---|
| **A** | Mixed GPU fleet (2 fast, 1 medium, 1 slow worker) | ~11 min: **pre-run before the meeting** | BNK keeps the slow GPU's queue short → TTFT p95 0.3 s vs 25–80 s |
| **B** | Same fleet, busy hour (75 % load) | ~12 min | p95 2.6 s vs 33–51 s |
| **C** | Identical GPUs, long 8k-token RAG context | ~6 min | best cache hit: 94 % vs 35–79 % |
| **D** | Identical GPUs, peak hour (100 % load) | ~5 min: **good live run** | p95 0.8 s vs 2.4–7.3 s |

Results from this Blueprint on a cold start (1 Oct 2026, 0 errors in every run):

![TTFT after preset A](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/ttft-A.png)

| Preset | Router | TTFT p50 | TTFT p95 | Cache hit |
|---|---|---|---|---|
| A | HAProxy | 243 ms | 78.9 s | 34 % |
| A | Istio + Dynamo EPP | 140 ms | 25.8 s | 61 % |
| A | Dynamo router | 130 ms | 27.6 s | 62 % |
| A | **BNK 2.4** | **108 ms** | **0.32 s** | **83 %** |
| D | HAProxy | 563 ms | 7.31 s | 34 % |
| D | Istio + Dynamo EPP | 175 ms | 2.82 s | 71 % |
| D | Dynamo router | 185 ms | 2.40 s | 69 % |
| D | **BNK 2.4** | **151 ms** | **0.76 s** | **88 %** |

![Which worker got the requests (preset A)](https://raw.githubusercontent.com/sokkolchen/bnk24-dynamo-demo/main/img/worker-split-A.png)

**What to say about the size of the gain:**

- **Against round robin, the gain is large.** Any KV-aware router, NVIDIA's included, gets most of it.
- **Against NVIDIA's own routers, the gain is real but smaller.** It shows mostly for the slowest users (p95/p99) and in cache hit.
- **For the typical user (p50), BNK is about even with NVIDIA's routers.** In preset C, BNK's p50 is 10–17 ms slower: the emulated DPU adds a GRE hop that a real BlueField would not.

Custom runs are possible: set *Pool*, *Prompt tokens*, *Cache level*, *Load* and *Answer tokens* in the bar at the top, then click **Run with my settings**. Limits are enforced.

**Other dashboard:** *BACKUP (classic)* is the original 3-router PoC layout. Keep it as a fallback; demo from **★ MAIN**.

---

## 4. Access and credentials

| What | Where | Login |
|---|---|---|
| Grafana (demo) | node1 → Access → **GRAFANA** | none needed to view/run. Admin: `admin` / `Junct10n!` (only to edit dashboards) |
| Demo API (called by the buttons) | jumphost → Access → **DEMO API** | none |
| Prometheus | node1 → Access → **PROMETHEUS** | none |
| Jumphost shell | jumphost → Access → **Web Shell** or SSH | your UDF key |

On the jumphost, `~/README.md` explains the files and scripts, and `~/CLAUDE.md` gives the same context for an AI coding assistant.

---

## 5. If something looks wrong

| Symptom | What to do (jumphost Web Shell) |
|---|---|
| Lab tile yellow for more than 20 min | `tail ~/lab-boot.log` shows the step it waits for |
| Lab tile red (**Burnt**) right after start | `bash ~/lab/lab-boot.sh` (~30 s once pods are up) |
| Before an important demo | `bash ~/lab/path-check.sh` → last line must be `PATH CHECK: OK` |
| A router shows errors in a run | `bash ~/lab/path-check.sh`; the red line says what to run |
| Still broken | normal **Restart** of the deployment (not Force Stop), wait 15 min |

---

*Lab facts:*
- UDF, 9 VMs.
- Kubernetes 1.35.
- BNK 2.4.0 with F5 EPP. It uses a 30-day evaluation licence, which renews automatically at start.
- NVIDIA Dynamo 1.5 mocker workers.
- Istio 1.29 with the Gateway API Inference Extension.
- MetalLB and VyOS routers.
