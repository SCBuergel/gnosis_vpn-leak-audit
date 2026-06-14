# A reality check on Gnosis VPN's static routing

**How well does a static, connection-time peer-exception list cover a HOPR node's
dynamically-discovered peer set — and what does the mismatch cost?**

*Internal report — Qubes AppVM `A-dev-full`, kernel 6.18, 2026-06-14. Companion tooling
in [`bin/`](bin); supporting notes in [`docs/`](docs).*

---

Gnosis VPN on Linux currently routes with what the project calls **static routing**:
on connect it selects a set of high-quality HOPR peers and installs host routes that
**except** them from the tunnel (they egress directly via the underlay), while all
other destinations fall through a catch-all into the tunnel. This report measures how
that design fares against the reality that a HOPR node's peer set is *discovered at
runtime* and *changes continuously*. We find the exception set reliably serves the
curated peers but covers only a minority of the node's actual peer dials; the
remainder are routed through the tunnel, where the node's own mixnet connections
cannot complete, producing a persistent unproductive traffic floor.

## 1. Background: how static routing works

Two route layers coexist on the host while the VPN is up:

```
# Catch-all: everything into the tunnel (these /1 routes beat the eth0 default
# by longest-prefix match):
0.0.0.0/1    dev wg0_gnosisvpn
128.0.0.0/1  dev wg0_gnosisvpn
default via 10.138.25.42 dev eth0

# Static exception set: ~22 /32 routes for selected high-quality peers, sent out
# the underlay directly:
34.159.20.69   via 10.138.25.42 dev eth0
35.242.223.88  via 10.138.25.42 dev eth0
...
```

Independently, the HOPR node process (`gnosis_vpn-worker`, uid 997) continuously
dials other HOPR nodes on `tcp/9091` to maintain the mixnet. The route layers decide,
per dial, whether it leaves via the underlay (`eth0`) or via the tunnel
(`wg0_gnosisvpn`):

- **Excepted peer** (in the static `/32` set) → egress `eth0` → reaches the peer.
- **Non-excepted peer** → catch-all → egress `wg0_gnosisvpn`. For the node's *own*
  mixnet dials this is self-referential (routing a HOPR connection through the HOPR
  tunnel it provides); the connection cannot bootstrap.

We use neutral terms throughout: a dial sourced from the underlay address is
**excepted**; one sourced from the tunnel address is **tunnelled**. (The tooling
labels these `HEALTHY`/`LEAKED` for historical reasons; read them as
excepted/tunnelled.)

## 2. Hypothesis

> A static exception list fixed at connection time cannot track a peer set that is
> discovered dynamically and changes over time. Therefore a substantial, time-varying
> fraction of the node's peer dials will be tunnelled rather than excepted, and those
> dials — being HOPR-through-HOPR — will fail to connect, imposing a persistent
> background traffic cost.

Testable predictions:

| | Prediction |
|---|---|
| **P1** | At any instant, a non-trivial fraction of `:9091` dials are tunnelled (sourced from the tunnel IP), not excepted. |
| **P2** | The excepted set is small and **stable**; the tunnelled set is larger and **churns** over time. |
| **P3** | Excepted dials reach `ESTABLISHED`; tunnelled dials remain `SYN-SENT` (never connect). |
| **P4** | Tunnelled dials produce a measurable idle traffic floor on the tunnel from SYN retransmissions. |
| **P5** | A peer's classification is determined by exception-set membership, not by anything time-varying, so steady-state state is fixed (no transitions). |

## 3. Methodology

### 3.1 Constraints and instruments

No packet capture was available (`tcpdump` is not installed on the host), so the study
relies on socket-table introspection and kernel counters:

- **`ss`** — enumerate the worker's `:9091` sockets with source/destination/state.
- **`/sys/class/net/wg0_gnosisvpn/statistics/{rx,tx}_packets`** — aggregate tunnel
  packet counts.
- **`iptables` accounting / owner-matching** — attribute and, in the control
  condition, suppress the worker's tunnel egress.

Four purpose-built tools (see Appendix A) implement the measurements:
`gnosis-vpn-block-peers` (instantaneous classification + experimental control),
`gnosis-vpn-track` (1 Hz longitudinal sampler), `gnosis-vpn-timeline` (swimlane
renderer), and `gnosis-vpn-demo` (packet accounting, gateway health-check excluded).

### 3.2 Classification by source IP

The classifier is the **source address the kernel selected** for each `:9091` socket.
Source address follows from egress interface, which follows from the routing decision:
a socket sourced from the underlay address (`10.137.0.30`) was excepted; one sourced
from the tunnel address (e.g. `10.128.0.60`) was tunnelled. The decision is confirmed
independently with `ip route get <peer>` (Appendix A). The tunnel address is derived
live each sample because WireGuard reassigns it across reconnects (observed
`.54 → .90 → .60 → .59 → .134`); a stale value would misclassify every socket.

### 3.3 Sampling and metrics

The node was sampled at **1 Hz** in steady state and across a deliberate
disconnect/reconnect. Per peer IP we record first-seen time, state at first sight,
per-sample state, presence, and transitions. Reported metrics: **coverage** (fraction
of distinct dialled peers that are excepted), **stability** (samples-present per
peer), **connectivity outcome** (TCP state by class), **churn** (set overlap between
snapshots), and **idle floor** (tunnel packets/s, with the worker's tunnel egress as
the experimental control).

## 4. Results

### 4.1 Instantaneous classification (P1, P3)

A representative snapshot immediately after a fresh connect (worker `gnosis_vpn-worker`,
pid 1827; all dials same process, same port):

| Peer IP | In exception set | Egress source | TCP state |
|---|---|---|---|
| `34.185.189.141` | yes | `10.137.0.30` (eth0) | **ESTABLISHED** |
| `35.242.223.88`  | yes | `10.137.0.30` (eth0) | **ESTABLISHED** |
| `35.198.136.5`   | no  | `10.128.0.60` (tunnel) | SYN-SENT |
| `34.89.170.79`   | no  | `10.128.0.60` (tunnel) | SYN-SENT |
| `34.179.176.136` | no  | `10.128.0.60` (tunnel) | SYN-SENT |
| `34.89.169.146`  | no  | `10.128.0.60` (tunnel) | SYN-SENT |
| … (7 tunnelled total) | no | tunnel | SYN-SENT |

The only differentiator between connecting and stuck is exception-set membership.
Excepted → `ESTABLISHED`; tunnelled → `SYN-SENT` with an unacknowledged SYN,
retransmitting. **P1 and P3 hold.**

### 4.2 Temporal coverage and churn (P2)

A 45-second 1 Hz capture in steady state. Each column is one second; `=` excepted &
present, `#` tunnelled & present, blank = no socket to that peer at that instant
(Figure 1, verbatim tool output):

```
Figure 1 — peer presence/state over 45 s (1 col = 1 s)

34.179.176.136  |#####         ##########          #########  |  x24
34.179.192.95   |#########         ##########          #######|  x26
34.185.189.141  |=============================================|  x45
35.242.223.88   |=============================================|  x45
35.246.241.197  |#####         ##########          #########  |  x24
34.141.57.130   |     #########    ##########      #########  |  x28
34.89.169.146   |     #########    ##########      #########  |  x28
34.89.170.79    |     #########    ##########      #########  |  x28
35.198.136.5    |     #########    ##########      #########  |  x28
34.141.68.157   |         #########          ##########       |  x19
```

Of **10 distinct peers** dialled in the window, **2 (20%) were excepted** and present
in **all 45 samples** (solid `=` rows — the curated, stable set); the other **8 (80%)
were tunnelled** and **intermittent** (speckled `#` rows, blinking in and out as the
worker rotates through non-excepted peers). Two snapshots taken 8 s apart shared only
1 of ~5 tunnelled peers (Jaccard ≈ 0.1), confirming rapid turnover. **P2 holds:**
coverage is a minority, the excepted set is stable, the tunnelled set churns.

### 4.3 State is fixed by membership, with a connection-time transient (P5)

In steady state, no peer transitions between excepted and tunnelled (`transitions = 0`
for every peer across the window) — classification is a function of static-list
membership, exactly as P5 predicts.

The one exception is a transient at connection time. Across a reconnect, **all** peers
(including normally-tunnelled ones) first appear **excepted** for ~1–2 s, then the
non-listed peers flip to **tunnelled** and stay there:

```
[+15s] UP    tunnel 10.128.0.59
[+15s] NEW   34.141.57.130   HEALTHY      # appears excepted at first…
[+17s] CHG   34.141.57.130   HEALTHY -> LEAKED   # …then flips once catch-all applies
[+26s] CHG   34.89.170.79    HEALTHY -> LEAKED
[+27s] CHG   34.89.169.146   HEALTHY -> LEAKED
```

This is consistent with the catch-all `/1` routes being installed a moment *after* the
interface appears: during that window the `eth0` default still applies to every dial.
It is a startup artifact, not steady-state behaviour, but it explains why a single
early snapshot can understate the tunnelled fraction.

### 4.4 Traffic cost (P4)

With no user applications running, the tunnel carried a steady idle floor of
**~25 packets/s** (~125 packets / 5 s), asymmetric toward TX — consistent with
continuous SYN retransmissions to unreachable tunnelled peers. Using the worker's
tunnel egress as an experimental control (dropping uid-997 packets on
`wg0_gnosisvpn`), the floor fell to ~52 packets / 5 s, the residual being the VPN's
own ~0.1 Hz gateway health-check plus this session's application keepalives. The
attributable cost of tunnelled dials is therefore the ~25 pkt/s majority of the idle
floor. **P4 holds.**

## 5. Discussion

All five predictions are supported on this host. The mechanism is structural rather
than incidental: a HOPR node's peer set is open-ended and discovered at runtime
(gossip/DHT), whereas the exception set is a finite snapshot fixed at connect. Any peer
discovered after connect, or simply not selected into the curated set, is routed into
the tunnel — and for the node's own mixnet dials that path cannot complete. Static
routing thus does its intended job well for the high-quality peers it targets (they
bypass reliably and connect), but by construction it cannot cover the long tail of the
dynamic peer set; here it covered ~20%, leaving an ~80% majority tunnelled and a
persistent unproductive packet floor.

The cleaner invariant is *process-scoped* rather than *destination-scoped*: a VPN
provider's own control-plane traffic should never egress the tunnel it provides,
regardless of which peers exist. A uid/cgroup-scoped policy route expressing "the
worker always uses the underlay" covers the full dynamic peer set in one rule and lets
those dials actually connect. Direction and a worked example are in
[`docs/routing-bug.md`](docs/routing-bug.md); the demo-grade suppression used as the
control here is in [`docs/clean-baseline.md`](docs/clean-baseline.md).

## 6. Threats to validity

- **Single host / single exit.** One Qubes AppVM, one VPN exit (UK/London). The
  *coverage ratio* in particular depends on this node's discovered peer set and the
  curated list of the day; the 20/80 split is illustrative, not a universal constant.
- **Inferred membership.** Exception-set membership is inferred from source IP and
  `ip route get`, not read from VPN config; the two agree, but we did not instrument
  the daemon's selection logic or its notion of "peer quality."
- **1 Hz sampling.** Sub-second sockets can be missed; presence counts are lower
  bounds. Aggregate `/sys` counters (which miss nothing) corroborate the floor.
- **Connectivity outcome.** "Never connects" is observed as persistent `SYN-SENT`
  over the observation window; we did not exhaustively rule out eventual success.
- **Non-persistent control.** All instrumentation is transient `iptables`/route state
  (QubesOS AppVM, gone on reboot); no host configuration was permanently changed.

## 7. Conclusion

Static routing reliably serves the curated high-quality peers it is designed for, but
on this host it covered only a minority (~20%) of the node's actual peer dials. The
dynamically-discovered majority were routed through the tunnel, could not complete, and
sustained a ~25 pkt/s idle floor. The gap is inherent to matching a static,
connection-time snapshot against a runtime-dynamic peer set. Promising directions are
periodic refresh of the exception set or — more robustly — a process-scoped routing
invariant for the worker, which would cover the entire dynamic peer set by
construction.

---

## Appendix A: Artifacts and reproduction

Tools (in [`bin/`](bin); need the tunnel up; `ss`/route reads need no root,
`iptables` accounting/control needs passwordless `sudo -n`):

| Tool | Role in the study |
|---|---|
| `gnosis-vpn-block-peers` | §4.1 instantaneous classification (`show`); §4.4 experimental control (`block`/`unblock`/`status`) |
| `gnosis-vpn-track [int] [dur]` | §4.2–4.3 1 Hz longitudinal sampler; writes a per-second snapshot TSV |
| `gnosis-vpn-timeline [tsv] [w]` | §4.2 swimlane renderer (Figure 1) from a track snapshot |
| `gnosis-vpn-demo [test N]` | §4.4 packet accounting, gateway health-check excluded |

```bash
# §4.1  classify the worker's current dials (excepted vs tunnelled)
bin/gnosis-vpn-block-peers show

# §4.2  capture 45 s at 1 Hz, then render the timeline
bin/gnosis-vpn-track 1 45
bin/gnosis-vpn-timeline

# §4.4  attribute the idle floor (control = drop the worker's tunnel egress)
IF=wg0_gnosisvpn
read a < <(cat /sys/class/net/$IF/statistics/tx_packets); sleep 5
read b < <(cat /sys/class/net/$IF/statistics/tx_packets); echo "$((b-a)) tx in 5s"
bin/gnosis-vpn-block-peers block      # re-measure; bin/gnosis-vpn-block-peers unblock

# confirm the routing decision behind a classification
ip route get <tunnelled-peer-ip>   # -> dev wg0_gnosisvpn
ip route get <excepted-peer-ip>    # -> via 10.138.25.42 dev eth0
```

**Host assumptions.** Tunnel iface `wg0_gnosisvpn`; in-tunnel gateway `10.128.0.1`;
worker uid `997`; eth0 `10.137.0.30/32` via `10.138.25.42`. The tunnel IP is not
stable — always derived live, never hardcoded. Override per host without editing the
scripts:

| Env var | Default | Used by |
|---|---|---|
| `GVPN_IF` | `wg0_gnosisvpn` | all |
| `GVPN_GW` | `10.128.0.1` | `gnosis-vpn-demo` |
| `GVPN_WORKER_UID` | `997` | `gnosis-vpn-block-peers` |

## Appendix B: Supporting documents

- [`docs/routing-bug.md`](docs/routing-bug.md) — detailed mechanism write-up and a
  process-scoped routing fix (intended for the Gnosis/HOPR team).
- [`docs/clean-baseline.md`](docs/clean-baseline.md) — operational runbook for the
  clean-baseline measurement and the experimental control used in §4.4.
