# Getting a clean packet baseline on the Gnosis VPN tunnel — runbook

Goal: measure only the packets **I deliberately** send/receive over `wg0_gnosisvpn`,
so a `ping` shows ~1:1 against a quiet baseline. Three different noise sources had
to be handled, each a *different* way: one **quit**, one **blocked**, one **ignored**.

> Host facts: iface `wg0_gnosisvpn`; VPN server gateway inside tunnel = `10.128.0.1`;
> worker runs as uid **997** (`gnosisvpn`); my shell/Claude run as uid **1000**;
> passwordless `sudo -n` works. The tunnel IP **drifts** across reconnects
> (seen `.54` → `.90` → `.20`) — always derive it, never hardcode:
> `TUN=$(ip -4 -o addr show wg0_gnosisvpn | awk '{print $4}' | cut -d/ -f1)`

## TL;DR — the clean-setup recipe

```bash
IF=wg0_gnosisvpn; GW=10.128.0.1

# 1. BLOCK the HOPR worker's self-looping peer dials (uid 997 into the tunnel).
bin/gnosis-vpn-block-peers block        # or the raw rule below:
# sudo iptables -A OUTPUT -o $IF -m owner --uid-owner 997 -j DROP

# 2. IGNORE the VPN gateway health-check: count real traffic separately from the
#    ICMP to/from the gateway (RETURN-only = counts but blocks nothing).
sudo iptables -N WGCNT_OUT 2>/dev/null; sudo iptables -F WGCNT_OUT
sudo iptables -A WGCNT_OUT -p icmp -d $GW -m comment --comment WGHC_OUT   -j RETURN
sudo iptables -A WGCNT_OUT          -m comment --comment WGREAL_OUT -j RETURN
sudo iptables -N WGCNT_IN  2>/dev/null; sudo iptables -F WGCNT_IN
sudo iptables -A WGCNT_IN  -p icmp -s $GW -m comment --comment WGHC_IN    -j RETURN
sudo iptables -A WGCNT_IN           -m comment --comment WGREAL_IN  -j RETURN
sudo iptables -C OUTPUT -o $IF -j WGCNT_OUT 2>/dev/null || sudo iptables -A OUTPUT -o $IF -j WGCNT_OUT
sudo iptables -C INPUT  -i $IF -j WGCNT_IN  2>/dev/null || sudo iptables -A INPUT  -i $IF -j WGCNT_IN

# 3. QUIT Claude Code (and any browser / internet app) — its keepalives are uid 1000.

# 4. Measure. The demo script does steps 2 + 4 for you:
bin/gnosis-vpn-demo test 20      # ~0 baseline, then ~20:20 on deliberate pings
```

Read the "real" counters directly any time:
```bash
sudo iptables -nvxL WGCNT_OUT   # WGREAL_OUT row = real packets out; WGHC_OUT = health-check out
sudo iptables -nvxL WGCNT_IN    # WGREAL_IN  row = real packets in;  WGHC_IN  = health-check in
```

## The three noise sources and why each is handled differently

| # | Source | Who | Why it hit the tunnel | Handling | Why that way |
|---|--------|-----|-----------------------|----------|--------------|
| 1 | HOPR peer self-loop | `gnosis_vpn-worker` (uid 997) | Dials peers on tcp/9091; peers not in the static `/32` bypass list fall through the `0.0.0.0/1` catch-all and route **into the tunnel** → permanent `SYN-SENT` → SYN retransmits (~25 pkt/s) | **BLOCK** (`-j DROP`) | The packets are useless (can never connect — HOPR-through-HOPR). Dropping them before they hit `wg0` removes the traffic *and* the wasted retransmits. Safe: the worker's legit traffic uses eth0, not the tunnel. |
| 2 | Tunnel health-check | `gnosis_vpn-root` (uid 0) | Spawns `ping -c1 -t6 10.128.0.1` every ~10s to confirm the tunnel/server is alive | **IGNORE** (count separately, don't block) | This is the VPN doing its job. Blocking it would make the daemon think the tunnel is dead → reconnect storms. So leave it flowing and just exclude it from the numbers. |
| 3 | App keepalives | `claude` etc. (uid 1000) | Persistent HTTPS connections (Anthropic API, telemetry, GitHub) send TCP keepalive PINGs through the catch-all routes | **QUIT** the app | It's real, deliberate-ish traffic from a real app; the only honest way to zero it is to not run the app during the demo. |

Root cause shared by #1 and #3: the host routes **all** internet traffic through the
tunnel via `0.0.0.0/1` + `128.0.0.0/1 dev wg0_gnosisvpn` (these `/1` routes beat
`default via eth0`). So anything that talks to the internet becomes HOPR packets
unless it has a `/32` bypass route via `10.138.25.42 dev eth0`.

## How each source was identified (so I can re-find them if they change)

- **What's egressing right now (TCP):** `sudo ss -tnp "src $TUN"` — attributes sockets to processes. Misses UDP/ICMP and SYN-SENT.
- **The worker loop (#1):** showed as `SYN-SENT 10.128.0.90:9091 -> <peer>:9091` owned by `gnosis_vpn-work`, sourced from the **tunnel IP**. Working peers are sourced from **eth0**. Classifier = source IP. See `docs/routing-bug.md`.
- **The health-check (#2):** invisible to `ss` snapshots because the `ping` socket is **ephemeral** (lives ~1s per 10s cycle). Found by **logging** egress: `sudo iptables -A OUTPUT -o $IF -m owner ! --uid-owner 1000 -j LOG --log-prefix "X "` then `sudo dmesg` → saw `ICMP TYPE=8 DST=10.128.0.1 TTL=6`. Pinned the process with a tight `pgrep -x ping` spin reading `/proc/<pid>/stat` for the parent → `gnosis_vpn-root`.
- **The app keepalives (#3):** `ss -tno "src $TUN"` showed `timer:(keepalive,...)` countdowns on `claude`'s sockets.

General lesson: **point-in-time `ss` misses short-lived/periodic flows.** To catch a
~10s blip, either log packets (`iptables -j LOG` → `dmesg`) or spin-poll
(`/proc/net/icmp`, `pgrep`) — don't trust a single snapshot.

## Verifying it's clean

```bash
bin/gnosis-vpn-demo check     # after quitting Claude: should say "(none ...)"
bin/gnosis-vpn-demo test 20   # rest ~0 real pkts; deliberate ~20 out / ~20 in; hc excluded
```
- `rest: real_out=0 real_in=0` (or near it) = clean.
- If rest is ~25 pkt/s → the worker DROP rule (step 1) isn't in place.
- If rest shows a 1-out/1-in blip every ~10s in the **hc** column → that's expected (the health-check, correctly excluded from real).

## Teardown (all of this is non-persistent — gone on reboot anyway)

```bash
bin/gnosis-vpn-demo counters-off                                   # removes the accounting chains
sudo iptables -D OUTPUT -o wg0_gnosisvpn -m owner --uid-owner 997 -j DROP   # removes the worker block
```

## If I want a *real fix* instead of a demo workaround

The worker self-loop (#1) is an actual misconfiguration — a VPN process must never
egress its own tunnel. The proper fix (lets those peers actually **connect**, not just
get dropped) is a uid-scoped policy route forcing the worker out eth0:

```bash
sudo ip route add default via 10.138.25.42 dev eth0 table 5182
sudo ip rule add uidrange 997-997 lookup 5182 priority 1000
```
Full write-up for the HOPR team is in `docs/routing-bug.md`. The health-check (#2)
and app keepalives (#3) are not bugs — leave them be; just exclude/quit for measurement.
