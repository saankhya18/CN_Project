# Adaptive Communication Network with Failure Recovery

**Project 30 · Protocol: UDP · Mininet + Open vSwitch + OpenFlow 1.3 + Ryu**

A resilient UDP communication framework in which a Ryu SDN controller detects network
failures (link down **and** links that silently lose packets) and dynamically re-routes the
application traffic through an alternate path. The UDP application repairs the resulting
loss itself and keeps running without a restart.

![Topology](docs/diagrams/topology.png)

```
  BEFORE failure:  h1 -> s1 -> s2 -> s4 -> h4          (Dijkstra cost 2)
  mininet> link s1 s2 down          -> OFPT_PORT_STATUS -> Ryu re-routes in ~10 ms
  AFTER  failure:  h1 -> s1 -> s3 -> s4 -> h4          (cost 4)
  mininet> link s1 s2 up            -> LLDP re-discovers the link -> traffic reverts to s2
  mininet> degrade s2 s4            -> ports stay UP; the UDP server's quality reports make
                                       Ryu mark the path DEGRADED -> traffic moves to s3
```

---

## Contents

1. [Project title](#1-project-title) · 2. [Problem statement](#2-problem-statement) ·
3. [Objectives](#3-objectives) · 4. [Architecture](#4-architecture) ·
5. [Network topology](#5-network-topology) · 6. [Components](#6-components) ·
7. [Technologies](#7-technologies) · 8. [Installation](#8-installation) ·
9. [Running instructions](#9-running-instructions) · 10. [Controller instructions](#10-controller-instructions) ·
11. [Mininet instructions](#11-mininet-instructions) · 12. [UDP client/server instructions](#12-udp-clientserver-instructions) ·
13. [Failure simulation](#13-failure-simulation) · 14. [Recovery mechanism](#14-recovery-mechanism) ·
15. [Testing](#15-testing) · 16. [Performance methodology](#16-performance-methodology) ·
17. [Results](#17-results) · 18. [Baseline comparison](#18-baseline-comparison) ·
19. [Limitations](#19-limitations) · 20. [Future improvements](#20-future-improvements) ·
[Team information](#team--project-information)

Detailed documents: [architecture](docs/architecture.md) · [testing](docs/testing.md) ·
[test report](tests/results/integration_report.md) · [performance](docs/performance.md) ·
[demo script](docs/demo.md) · [viva Q&A](docs/viva.md) · [troubleshooting](docs/troubleshooting.md) ·
[rubric gap analysis](docs/rubric_gap_analysis.md) · [self-evaluation](docs/evaluation.md)

### Repository structure

```
adaptive-communication-network/
├── README.md
├── requirements.txt / requirements-ryu.txt / build-constraints.txt
├── config/network_config.json        single source of truth: hosts, IPs, MACs, ports, costs, delays, feedback settings
├── topology/adaptive_topology.py     Mininet topology + CLI commands (paths, flows, ctrl, events, quality, degrade, apps)
├── controller/
│   ├── adaptive_controller.py        Ryu app: discovery, failure detection, app feedback, Dijkstra routing, REST
│   └── path_computation.py           Dijkstra (pure Python, unit-tested)
├── application/
│   ├── protocol.py                   AUP wire format: DATA / ACK / NACK / FIN / FIN_ACK / REPORT
│   ├── udp_server.py                 receiver: gap detection, NACK, statistics, quality reports to the controller
│   └── udp_client.py                 sender: rate control, retransmission buffer, path liveness
├── scripts/                          install, start_controller, start_network, mx (run on a host), run_demo,
│                                     watch_flows, run_tests, run_experiment, cleanup
├── experiments/
│   ├── harness.py                    shared test bed (controller + Mininet + apps + link manipulation)
│   ├── run_experiments.py            automated static / dynamic_nofb / dynamic experiments
│   ├── metrics.py                    metric definitions computed from raw packet files
│   ├── analyze_results.py            summary tables + graphs
│   └── results/                      runs.csv, summary.md/.csv, plots/   (raw/ is git-ignored)
├── tests/
│   ├── test_path_computation.py, test_protocol.py      unit + loopback tests (no Mininet)
│   ├── integration/run_integration_tests.py             automated Mininet tests T1–T8
│   └── results/                      integration_report.md / .json (actual results)
├── docs/                             architecture, testing, performance, demo, viva, troubleshooting,
│   ├── diagrams/                     topology.png, architecture.png (+ make_diagrams.py)
│   └── ...                           rubric_gap_analysis.md, evaluation.md
├── logs/                             demo logs (generated)
└── screenshots/README.md             checklist of screenshots for the report
```

---

## 1. Project title

**Adaptive Communication Network with Failure Recovery** (Project 30, UDP).

## 2. Problem statement

*Develop a resilient UDP communication framework in which the SDN controller detects network
failures and dynamically reroutes application traffic through alternate paths.*

UDP gives no delivery guarantee and no feedback. A network with static routes keeps forwarding
into a dead link until a human repairs it, and nothing in the network notices a link that
stays up but drops packets. We combine two layers of resilience that cooperate:

* **Network layer (SDN):** Ryu sees every switch and link, detects failures, recomputes paths
  and reprograms the Open vSwitches within milliseconds.
* **Application layer (UDP):** the application numbers its packets, detects and repairs the
  loss that a failover causes, keeps streaming, and reports path quality back to the
  controller so the network can react to problems that only the application can see.

## 3. Objectives

| # | Required functionality | Where it is implemented | Proven by |
|---|---|---|---|
| 1 | Implement UDP communication | `application/udp_client.py`, `udp_server.py`, `protocol.py` | unit tests, T1 |
| 2 | Create multiple network paths | `config/network_config.json`, `topology/adaptive_topology.py` | T1–T3 (`paths`) |
| 3 | Monitor topology and link status | LLDP (`_link_add/_link_delete`), PORT_STATUS (`_port_status`), flow statistics (`_monitor`), app quality reports (`_handle_report`) | T2, T4 |
| 4 | Simulate link/path failures | `link s1 s2 down`, `degrade s2 s4`, `harness.link_down/degrade` | T2–T6, experiments |
| 5 | Detect failures dynamically | PORT_STATUS (ms), LLDP timeout (s), application reports (~1 s) | T2, T4 |
| 6 | Reroute application traffic | `_recompute_routes()` → Dijkstra → `_install_path()` | T2–T6, OpenFlow counters |
| 7 | Application-level recovery | sequence numbers, ACK, NACK, retransmission buffer with deadline, PATH_DOWN/RESTORED, join-mid-stream | T3, T6, T7, unit tests |
| 8 | Measure recovery time, packet loss, latency | `experiments/metrics.py`, `run_experiments.py`, `analyze_results.py` | `experiments/results/` |
| + | Compare with a baseline | `ADAPTIVE_MODE=static` and the ablation `dynamic_nofb` | T8, experiments |

## 4. Architecture

![Architecture](docs/diagrams/architecture.png)

```
  UDP application (h1-h3 clients, h4 server: socket/bind/sendto/recvfrom)
        │  DATA(seq, ts) ─►        ◄─ ACK / NACK          quality REPORT ─► 10.0.0.254:6000
        ▼                                                        │
  Mininet hosts ── veth ── Open vSwitch s1..s4 (OpenFlow 1.3, failMode=secure)
        │   prio 200 REPORT → CONTROLLER │ prio 100 ip,src,dst → output:N │ prio 0 → CONTROLLER
        ▼                                                        ▼
  Ryu controller: topology (LLDP) ─ failure detection (PORT_STATUS / LLDP / app report)
        ─► topology update ─► Dijkstra ─► FLOW_MOD (egress→ingress, delete stale) ─► alternate path
        ─► UDP application continues (gap → NACK → retransmission → final loss 0)
```

**Data flow:** UDP datagrams are forwarded by OVS using per-host-pair rules installed by Ryu.
Once the rules exist, data packets never visit the controller.
**Control flow:** OVS → Ryu: PORT_STATUS, PACKET_IN (ARP, first packets, quality reports),
LLDP, FLOW_STATS. Ryu → OVS: FLOW_MOD, PACKET_OUT.
Full explanation with sequence diagrams: [docs/architecture.md](docs/architecture.md).

## 5. Network topology

| Host | IP | MAC | Attached to | Role |
|------|----|-----|-------------|------|
| h1 | 10.0.0.1/24 | 00:00:00:00:00:01 | s1 port 1 | UDP client 1 |
| h2 | 10.0.0.2/24 | 00:00:00:00:00:02 | s1 port 2 | UDP client 2 |
| h3 | 10.0.0.3/24 | 00:00:00:00:00:03 | s1 port 3 | UDP client 3 |
| h4 | 10.0.0.4/24 | 00:00:00:00:00:04 | s4 port 1 | UDP server, port 5000 |
| (virtual) | 10.0.0.254 | 00:00:00:00:00:fe | – | controller feedback address, UDP 6000 |

| Link | Ports | Cost | Delay (tc netem) | Role |
|------|-------|------|------------------|------|
| s1 – s2 | s1:4 ↔ s2:1 | 1 | 1 ms | primary |
| s2 – s4 | s2:2 ↔ s4:2 | 1 | 1 ms | primary |
| s1 – s3 | s1:5 ↔ s3:1 | 2 | 3 ms | backup |
| s3 – s4 | s3:2 ↔ s4:3 | 2 | 3 ms | backup |

Switch DPIDs are s1 = 1 … s4 = 4. Interface names follow the port numbers (`s1-eth4` = s1 port
4). The primary path costs 2 and the backup 4, so Dijkstra prefers s2 whenever it is usable.
The two paths share no inter-switch link. All three clients share the ingress switch s1, so
**all three flows experience every failure** (multi-client scenario). Everything comes from
`config/network_config.json`, which both the topology and the controller read.

> The assignment's example names the server `h2`. Here the server is `h4` so that three
> clients (h1–h3) can share the failure. The paths are exactly the required
> `h1 → s1 → s2 → s4 → server` and `h1 → s1 → s3 → s4 → server`.

## 6. Components

| Component | File | Responsibility |
|---|---|---|
| AUP protocol | `application/protocol.py` | 28-byte header `!2sBBBxHIdd` (magic, version, type, flags, client_id, seq, ts1, ts2); DATA, ACK (+cumulative ack), NACK (≤ 256 seqs), FIN/FIN_ACK, REPORT (JSON) |
| UDP client | `application/udp_client.py` | non-blocking socket + `select`; exact-rate sending; retransmission buffer with a 2 s deadline; NACK handling; RTT; PATH_DOWN after 250 ms without ACK, PATH_RESTORED; FIN handshake; CSV/JSON results |
| UDP server | `application/udp_server.py` | single-threaded `select` event loop serving any number of clients (demultiplexed by client_id + address); ACK every packet; gap detection; NACK retries (100 ms × 15); one-way latency, RFC 3550 jitter, losses; joins a stream mid-way after a restart; quality reports every 0.5 s; counters for malformed datagrams and send errors |
| Topology | `topology/adaptive_topology.py` | config-driven Mininet topology, RemoteController, OF1.3, `failMode=secure`, optional userspace datapath; CLI commands `paths`, `flows`, `ctrl`, `events`, `quality`, `degrade`, `undegrade`, `apps`, `stopapps` |
| Controller | `controller/adaptive_controller.py` | LLDP discovery, PORT_STATUS/LinkDelete failure detection, app-feedback handler, proxy ARP, proactive rule installation, stale-rule deletion, revert, flow statistics, REST API, 3 modes |
| Path computation | `controller/path_computation.py` | Dijkstra with deterministic tie-breaking |
| Test bed | `experiments/harness.py` | starts controller + topology, drives links (`link_down`, `degrade`), observes REST and OpenFlow counters |
| Experiments | `experiments/run_experiments.py`, `metrics.py`, `analyze_results.py` | automated measurement campaign, metrics, tables and graphs |
| Tests | `tests/` | 18 unit/loopback tests + 8 automated Mininet integration tests |

## 7. Technologies

| Component | Version used / tested |
|-----------|----------------------|
| OS | Ubuntu 20.04 / 22.04 (validated end-to-end on 24.04) |
| Mininet | 2.3.0 (apt package `mininet`) |
| Open vSwitch | 2.13+ (apt `openvswitch-switch`; validated with 3.3) |
| OpenFlow | 1.3 |
| SDN controller | Ryu 4.34 in a Python 3.9 virtual environment `.venv-ryu` |
| Application | Python 3 standard library only: `socket`, `select`, `struct`, `json` |
| Analysis | Python 3 + matplotlib |

Ryu runs in its own Python 3.9 environment because Ryu 4.34 needs eventlet 0.30.2, which
crashes on Python ≥ 3.10 (`TypeError: cannot set 'is_timeout' attribute of immutable type
'TimeoutError'`). `scripts/install.sh` uses `uv` to create that venv with pinned, verified
versions, so it works on every Ubuntu release.

## 8. Installation

You need Ubuntu 20.04/22.04 (VM recommended, 2+ vCPUs), sudo, and internet access.

```bash
git clone <your-repo-url> adaptive-communication-network      # or unzip the archive
cd adaptive-communication-network
bash scripts/install.sh            # apt: mininet, openvswitch-switch, matplotlib ...; Ryu venv
sudo mn --test pingall             # verify Mininet + OVS: "Results: 0% dropped (2/2 received)"
sudo mn -c                         # clean up
python3 -m unittest discover -s tests     # 18 tests, no Mininet needed
```

Interactive Mininet check from the setup guide: `sudo mn`, then `mininet> pingall`,
`mininet> exit`, `sudo mn -c`.

What `install.sh` does, step by step:

```bash
sudo apt-get update
sudo apt-get install -y mininet openvswitch-switch openvswitch-testcontroller iperf iputils-ping ethtool net-tools curl git python3-matplotlib
sudo systemctl enable --now openvswitch-switch
sudo systemctl disable --now openvswitch-testcontroller 2>/dev/null || true   # frees TCP 6653 for Ryu
curl -LsSf https://astral.sh/uv/install.sh | sh                                 # ~/.local/bin/uv
~/.local/bin/uv python install 3.9
~/.local/bin/uv venv -p 3.9 .venv-ryu
~/.local/bin/uv pip install --python .venv-ryu -r requirements-ryu.txt --build-constraint build-constraints.txt
.venv-ryu/bin/ryu-manager --version          # -> ryu-manager 4.34
```

## 9. Running instructions

| Terminal | Command | Purpose |
|---|---|---|
| T1 | `./scripts/start_controller.sh dynamic` | Ryu controller (log shows every event) |
| T2 | `./scripts/start_network.sh` | Mininet topology + CLI (`--userspace` on WSL2/containers, `--no-delay` without netem) |
| T3 | `./scripts/mx.sh h4 python3 -u application/udp_server.py --port 5000 --out-dir logs/demo` | UDP server on h4 |
| T4 | `./scripts/mx.sh h1 python3 -u application/udp_client.py --server 10.0.0.4 --client-id 1 --rate 100 --duration 300 --out-dir logs/demo` | UDP client on h1 |
| T5 | `./scripts/watch_flows.sh` | live OpenFlow rules and packet counters |

Shortcut: `./scripts/start_network.sh --start-apps` starts the server and three clients itself
(logs in `logs/demo/`). The full 10-minute demo script with expected output is in
[docs/demo.md](docs/demo.md). Guided version: `./scripts/run_demo.sh`.

Cleanup: `exit` in Mininet, Ctrl-C the controller, then `./scripts/cleanup.sh` (runs `sudo mn -c`).

## 10. Controller instructions

```bash
./scripts/start_controller.sh dynamic        # proposed: link events + application feedback
./scripts/start_controller.sh dynamic_nofb   # ablation: link events only
./scripts/start_controller.sh static         # baseline: routes computed once, never changed
```

It runs `.venv-ryu/bin/ryu-manager --observe-links --ofp-tcp-listen-port 6653 --wsapi-port 8080
controller/adaptive_controller.py`. Events are also written to `logs/controller_events.csv`.

| Event | Meaning |
|---|---|
| `SWITCH_CONNECTED`, `LINK_UP` | discovery (OpenFlow handshake, LLDP) |
| `ROUTE_INSTALLED`, `REROUTE` | path (re)computed and rules sent; shows old and new path and the reason |
| `PORT_DOWN`, `LINK_DOWN`, `PORT_UP` | failure detection / restoration |
| `LINK_DEGRADED`, `DEGRADED_EXPIRED` | application reported loss / hold-down over |
| `NO_PATH` | no path exists for a host pair (old rules kept) |
| `STATIC_FROZEN`, `STATIC_NO_ACTION`, `APP_DEGRADED_IGNORED` | baseline / ablation behaviour |
| `[FLOW-STATS] … s1=600 s2=600 s3=0 s4=600` | packets forwarded per switch in the last 2 s |

REST API (read-only): `curl -s 127.0.0.1:8080/adaptive/{status,paths,topology,events,stats,quality}`.

## 11. Mininet instructions

| Command (Mininet CLI) | Shows / does |
|---|---|
| `net`, `nodes`, `links`, `dump` | Mininet topology |
| `pingall` | connectivity (ARP answered by the controller, IPv4 by installed rules) |
| `paths` | current path of every host pair (controller REST) |
| `flows` | the controller's route rules on s1..s4 with packet counters |
| `ctrl` | controller status (mode, switches, links, ready, degraded links) |
| `events 20` | last 20 controller events |
| `quality` | latest application quality reports and DEGRADED links |
| `link s1 s2 down` / `link s1 s2 up` | fail / restore a link |
| `degrade s2 s4 [loss%]` / `undegrade s2 s4` | lossy link with ports still up |
| `apps [rate] [size]` / `stopapps` | start / stop server + 3 clients |
| `sh ovs-ofctl -O OpenFlow13 dump-flows s1` | raw OpenFlow table |
| `sh ovs-vsctl show` | switch ↔ controller connections |

## 12. UDP client/server instructions

```bash
# inside Mininet:  h4 python3 application/udp_server.py ...   or from any terminal: ./scripts/mx.sh h4 python3 ...
python3 application/udp_server.py --port 5000 --out-dir logs/demo
python3 application/udp_client.py --server 10.0.0.4 --port 5000 --client-id 1 \
        --rate 100 --size 512 --duration 60 --out-dir logs/demo
```

| Server option | Default | Client option | Default |
|---|---|---|---|
| `--bind`, `--port` | 0.0.0.0, 5000 | `--server`, `--port`, `--client-id` | 10.0.0.4, 5000, 1 |
| `--nack-interval`, `--max-nacks` | 0.1 s, 15 | `--rate`, `--size`, `--duration` | 100 pps, 512 B, 30 s |
| `--quality-interval`, `--feedback` | 0.5 s, 10.0.0.254:6000 | `--ack-timeout` | 0.25 s |
| `--no-quality-report`, `--no-nack` | off | `--retx-max-age`, `--no-retx` | 2.0 s, off |
| `--join-threshold` | 100 | `--linger` | 3 s |
| `--duration`, `--report-interval`, `--out-dir`, `--verbose` | | `--report-interval`, `--out-dir` | |

Outputs: `server_packets.csv` (one row per packet: seq, send/orig/recv time, retx flag,
latency), `server_summary.json` (per client: sent, received, network/final loss, recovered,
latency avg/p95/max, jitter, max gap, goodput, NACKs), `client<N>_packets.csv`,
`client<N>_events.csv` (PATH_DOWN/RESTORED), and `client<N>_summary.json`.

## 13. Failure simulation

| Failure | Command | Detected by |
|---|---|---|
| Primary link down | `mininet> link s1 s2 down` (both interfaces `s1-eth4`, `s2-eth1` go down) | OFPT_PORT_STATUS, in ms |
| Other primary link | `mininet> link s2 s4 down` | OFPT_PORT_STATUS |
| Degraded link (gray failure) | `mininet> degrade s2 s4` (netem 10 % loss, or htb bottleneck without netem); ports stay up | application quality reports, ~1 s |
| Both paths cut | `link s1 s2 down` + `link s1 s3 down` | NO_PATH; the client's PATH_DOWN |
| Server crash | `kill -9` the server, start it again | client PATH_DOWN/RESTORED; server joins mid-stream |
| Repeated failures | repeat `link s1 s2 down/up` | every event handled |

All of these are automated in `tests/integration/run_integration_tests.py` (T2–T8) and,
except the double failure and the server crash, in `experiments/run_experiments.py`.

## 14. Recovery mechanism

**Network level (controller):**

1. **Detect:** PORT_STATUS with `OFPPS_LINK_DOWN` (`_port_status`). Backups: LLDP
   `EventLinkDelete`, and the application's quality reports (`_handle_report`).
2. **Update topology:** remove both directions of the link from `self.links`, or, for a
   degraded path, add a cost penalty of +10 for a 20 s hold-down.
3. **Recompute:** Dijkstra for all 12 host pairs over the live, penalised graph.
4. **Reprogram:** for each changed pair, `FLOW_MOD ADD` on the new path from egress to
   ingress, then `FLOW_MOD DELETE_STRICT` on switches that left the path.
5. **Restore:** when LLDP sees the link again (`EventLinkAdd`), or a hold-down expires,
   recompute and revert to the cheaper primary path.
6. **No path:** log `NO_PATH`, keep the old rules, and re-route as soon as a path returns.

**Application level:** sequence numbers expose gaps. The server NACKs missing packets
immediately and then every 100 ms (up to 15 times). The client retransmits packets younger than
2 s (a live-stream deadline). A cumulative ACK bounds the client's buffer. The client never
stops: no ACK for 250 ms gives PATH_DOWN, and the next ACK gives PATH_RESTORED with the outage
length. FIN carries the exact number sent, so tail losses are counted too.

**Socket → SDN:** every 0.5 s the server sends each flow's window loss to 10.0.0.254:6000. A
priority-200 OpenFlow rule hands it to Ryu as a PACKET_IN. Two consecutive reports with ≥ 5 %
loss mark the flow's path DEGRADED, which triggers step 2 above. See
[architecture §8](docs/architecture.md#8-socket--sdn-feedback-application-driven-re-routing).

## 15. Testing

```bash
python3 -m unittest discover -s tests -v       # 18 unit/loopback tests, ~15 s, no Mininet
./scripts/run_tests.sh                         # + automated Mininet tests T1–T8, ~6 min
```

| ID | Scenario | Validation run |
|---|---|---|
| T1 | Normal UDP communication | PASS |
| T2 | Primary path failure: detection and rule changes | PASS (detection 6.6 ms, re-route 8.0 ms) |
| T3 | Recovery through the backup path, then revert | PASS (recovery 23.8 ms, final loss 0) |
| T4 | Degraded link, app feedback ON | PASS (LINK_DEGRADED after 1.09 s; 0 % loss after re-route) |
| T4b | Degraded link, app feedback OFF (ablation) | PASS (loss persists: 90 %) |
| T5 | Repeated failure/recovery ×3 | PASS (mean recovery 22.1 ms, final loss 0) |
| T6 | No alternate path | PASS (NO_PATH, nothing crashes, recovers when s1-s3 returns) |
| T7 | Server crash + restart | PASS (joins mid-stream at seq 802, 0 NACK storm) |
| T8 | Static baseline | PASS (outage 3018 ms = failure length) |

Setup, commands, expected and **actual** results for each check:
[docs/testing.md](docs/testing.md) and [tests/results/integration_report.md](tests/results/integration_report.md).

## 16. Performance methodology

```bash
./scripts/run_experiment.sh --quick     # 15 runs, ~12 min
./scripts/run_experiment.sh             # 90 runs: 3 modes x 5 scenarios x {100,300} pps x 3 reps, ~70 min
```

* **Modes:** `static` (baseline), `dynamic_nofb` (link-event re-routing only), `dynamic` (proposed).
* **Scenarios** (seconds after the 3 clients start; 30 s streams, 512 B packets):
  `no_failure`; `short_failure` (s1-s2 down at 10 s for 2 s); `long_failure` (8 s, longer than
  the 2 s retransmission deadline); `repeated_failure` (down at 6, 13 and 20 s for 2 s each);
  `degraded_link` (s2-s4 lossy from 10 s to 25 s, ports up).
* **Per client and run** (`experiments/results/runs.csv`): timestamp, scenario, path before/
  during/after, packets sent / received, network and final loss %, loss during the event,
  average / p95 / maximum latency, jitter, recovery time, detection and re-route time,
  goodput (throughput), duration.
* Every number is computed from packet-level logs and the controller's event timestamps, which
  share one clock. Definitions: [docs/performance.md](docs/performance.md).

## 17. Results

<!-- RESULTS-START -->
All **90 runs** completed: 3 modes × 5 scenarios × {100, 300} pps × 3 repetitions, with 3 clients
per run. That gives n = 9 samples per cell (mean ± std). Full tables:
[experiments/results/summary.md](experiments/results/summary.md). Graphs: `experiments/results/plots/`.
Validation environment: Ubuntu 24.04, OVS **userspace** datapath, **no link delay** (so
latencies are sub-millisecond). Re-run `./scripts/run_experiment.sh` on your VM for your own numbers.

Key rows at 100 pps per client:

| Scenario | Mode | Network loss % | Final loss % | Recovery time (ms) | Detection (ms) | Re-route (ms) | Avg latency (ms) | Goodput (kbit/s) | Path during event |
|---|---|---|---|---|---|---|---|---|---|
| No failure | static | 0.00 | 0.00 | – | – | – | 0.70 | 409.1 | s1-s2-s4 |
| No failure | dynamic | 0.00 | 0.00 | – | – | – | 0.65 | 409.6 | s1-s2-s4 |
| Short failure (2 s) | static | 6.70 ± 0.03 | 0.06 ± 0.02 | 2014 ± 5 | 6.0 | – | 0.75 | 408.7 | s1-s2-s4 (dead) |
| Short failure (2 s) | dynamic | 0.07 ± 0.02 | **0.00** | **26.8 ± 3.3** | 7.0 | 10.3 | 0.73 | 409.4 | **s1-s3-s4** |
| Long failure (8 s) | static | 26.70 ± 0.02 | 20.07 ± 0.03 | 8016 ± 5 | 7.8 | – | 0.70 | 327.4 | s1-s2-s4 (dead) |
| Long failure (8 s) | dynamic | 0.05 ± 0.02 | **0.00** | **22.0 ± 3.2** | 5.6 | 7.0 | 0.69 | 409.6 | **s1-s3-s4** |
| Repeated (3 × 2 s) | static | 20.06 ± 0.03 | 0.16 ± 0.02 | 2016 ± 7 | 7.7 | – | 0.72 | 409.0 | s1-s2-s4 (dead) |
| Repeated (3 × 2 s) | dynamic | 0.17 ± 0.03 | **0.00** | **23.5 ± 3.9** | 7.2 | 10.7 | 0.71 | 409.6 | **s1-s3-s4** |
| Degraded link (15 s) | static | 41.41 ± 4.99 | 23.15 ± 9.87 | 14991 | – | – | 10.35 | 314.8 | s1-s2-s4 (lossy) |
| Degraded link (15 s) | dynamic | 2.58 ± 0.21 | **0.00** | **1127 ± 9** | 1122 (app report) | 1124 | 1.32 | 409.6 | **s1-s3-s4** |

The degraded link was emulated with an htb bottleneck at 50 % of the offered load, because the
validation kernel had no netem. Recovery time for a degraded link is the time until the last
lost packet. In the dynamic runs without a degradation, no false `LINK_DEGRADED` event occurred
(0 of 48 runs).
<!-- RESULTS-END -->

## 18. Baseline comparison

<!-- BASELINE-START -->
| Scenario (100 pps) | static (baseline) | dynamic_nofb (link events only) | dynamic (proposed) |
|---|---|---|---|
| Short failure: recovery / final loss | 2014 ms / 0.06 % | 20.4 ms / 0.00 % | 26.8 ms / 0.00 % |
| Long failure: recovery / final loss | 8016 ms / 20.07 % | 25.1 ms / 0.00 % | 22.0 ms / 0.00 % |
| Repeated failures: recovery / final loss | 2016 ms / 0.16 % | 24.1 ms / 0.00 % | 23.5 ms / 0.00 % |
| Degraded link: recovery / final loss | 14991 ms / 23.15 % | 14997 ms / 23.36 % | 1127 ms / 0.00 % |

* **Static vs SDN:** with static routing the outage lasts as long as the failure (2 s, 8 s). An
  8 s outage permanently loses ~20 % of the stream, because that data is older than the 2 s
  retransmission deadline. SDN re-routing restores traffic in ~20–27 ms whatever the failure
  length, and NACK recovery brings final loss to 0.
* **Link events only vs application feedback:** for hard link failures the two dynamic modes
  are the same (both use PORT_STATUS). For a degraded link with its ports up, link-event SDN is
  as helpless as static routing (≈ 40 % network loss, 23 % final loss for 15 s). The
  application's quality reports let Ryu re-route in ~1.1 s, giving 0 % final loss.
* The 300 pps results show the same pattern (see `summary.md`). More packets fall inside the
  failover window, but the recovery times are unchanged.

Graphs: `recovery_time.png`, `packet_loss.png`, `loss_during_event.png`, `degraded_recovery.png`,
`controller_timing.png`, and `timeline_<mode>_<scenario>_r100_rep1.png` for each mode.
<!-- BASELINE-END -->

## 19. Limitations

* Hosts are configured statically in `network_config.json` (no dynamic host discovery).
* The controller is a single point of failure. Existing rules keep forwarding if it dies, but
  there is no re-routing until it returns.
* Re-routing is reactive: a few packets per client are lost at each failover (recovered by
  NACK). OpenFlow fast-failover groups would remove even that but would hide the controller's
  role.
* Degraded-link detection needs about 1 s (2 report windows) and penalises the **whole path**,
  because end-to-end loss cannot be localised to one link. Thresholds are static.
* Only the server sends quality reports. If the server's own access link fails, its reports
  cannot reach the controller (the clients' PATH_DOWN still detects it).
* Path costs are administrative values, not measured load.
* Validation was done with the OVS userspace datapath and without link delays. Absolute
  latencies on a kernel-datapath VM will differ.

## 20. Future improvements

* OpenFlow fast-failover groups (`OFPGT_FF`) for sub-ms local repair, with the controller
  re-optimising afterwards.
* Per-link loss localisation using the per-rule flow counters already collected, so only the
  bad link is penalised.
* Load- and latency-aware costs; splitting clients across s2 and s3.
* Client-side reports and per-flow (instead of global) re-routing decisions.
* Controller redundancy (multiple controllers / ONOS clustering).
* FEC as an alternative to NACK for very latency-sensitive streams.

## Team / project information

| Field | Value |
|-------|-------|
| Project number | 30 |
| Title | Adaptive Communication Network with Failure Recovery |
| Course | `<course name / code>` |
| Institution | `<college name>` |
| Team members | `<name 1 (roll no.)>`, `<name 2 (roll no.)>`, `<name 3 (roll no.)>` |
| Guide / instructor | `<name>` |
| Academic year | `<2025-26>` |
| Repository | `<GitHub URL>` |
