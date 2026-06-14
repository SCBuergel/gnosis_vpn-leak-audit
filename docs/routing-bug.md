# Gnosis VPN node misroutes its own peer traffic into its own tunnel

**Reporter:** SCBuergel
**Date:** 2026-06-14
**Host:** Qubes AppVM (`A-dev-full`), kernel 6.18, eth0 = `10.137.0.30/32`
**Components:** `gnosis_vpn-root`, `gnosis_vpn-worker` (HOPR node), WireGuard iface `wg0_gnosisvpn`

## Summary

The Gnosis VPN node dials other HOPR peers (tcp/9091) and, for any peer whose IP
is **not** in a static `/32` bypass list, routes that dial **into the HOPR tunnel
the node itself provides**. The connection can never bootstrap (HOPR-through-HOPR),
so the socket sits in `SYN-SENT` and the kernel retransmits SYNs indefinitely. This
produces a permanent background traffic floor (~25–30 packets/sec while otherwise
idle) and wastes CPU/encryption on connections that can never succeed.

Because a HOPR node discovers peers **dynamically** while the bypass list is a
**static finite snapshot**, this is a structural mismatch, not a transient or a
single missing route.

## Environment / routing layout

The host routing table layers two things:

```
# Catch-all that sends everything into the tunnel (these /1 routes beat the
# eth0 default via longest-prefix match):
0.0.0.0/1   dev wg0_gnosisvpn proto static
128.0.0.0/1 dev wg0_gnosisvpn proto static
default via 10.138.25.42 dev eth0 onlink

# 22 static bypass routes for specific peer IPs, sending them out eth0 directly:
34.159.20.69    via 10.138.25.42 dev eth0 proto static
34.185.189.141  via 10.138.25.42 dev eth0 proto static
35.242.223.88   via 10.138.25.42 dev eth0 proto static
... (22 total /32-style entries)
```

The `gnosis_vpn-worker` process (runs as user `gnosisvpn`, uid 997) listens on
`0.0.0.0:9091` and continuously dials other HOPR nodes on `:9091`. Whether each
dial goes out eth0 or into the tunnel depends entirely on whether that peer's IP
is in the static bypass list.

## Evidence

### 1. Perfect correlation between bypass-list membership and connection success

Same process, same port, same kind of peer. The **only** differentiator is whether
the peer IP is in the static bypass list:

| Peer IP          | In `/32` bypass list? | Egress source        | TCP state        |
|------------------|-----------------------|----------------------|------------------|
| `34.185.189.141` | YES                   | `10.137.0.30` (eth0) | **ESTABLISHED**  |
| `35.242.223.88`  | YES                   | `10.137.0.30` (eth0) | **ESTABLISHED**  |
| `35.198.136.5`   | NO                    | `10.128.0.90` (tun)  | SYN-SENT (stuck) |
| `34.141.68.157`  | NO                    | `10.128.0.90` (tun)  | SYN-SENT (stuck) |
| `34.141.57.130`  | NO                    | `10.128.0.90` (tun)  | SYN-SENT (stuck) |

### 2. Socket attribution (`ss`, run as root)

```
# Working — sourced from eth0 (10.137.0.30):
ESTAB    0 0 10.137.0.30:9091  34.185.189.141:9091 users:(("gnosis_vpn-work",pid=1827,fd=40))
ESTAB    0 0 10.137.0.30:43224 35.242.223.88:9091  users:(("gnosis_vpn-work",pid=1827,fd=37))

# Stuck — sourced from the tunnel IP (10.128.0.90), permanently SYN-SENT:
SYN-SENT 0 1 10.128.0.90:9091  35.198.136.5:9091   users:(("gnosis_vpn-work",pid=1827,fd=34))
SYN-SENT 0 1 10.128.0.90:9091  34.141.68.157:9091  users:(("gnosis_vpn-work",pid=1827,fd=20))
SYN-SENT 0 1 10.128.0.90:9091  34.141.57.130:9091  users:(("gnosis_vpn-work",pid=1827,fd=42))
```

The stuck sockets report `unacked:1` (an unacknowledged SYN) and keep re-firing.

### 3. The kernel's routing decision confirms it

```
ip route get 34.185.189.141   # in list
  -> 34.185.189.141 via 10.138.25.42 dev eth0 src 10.137.0.30   # bypasses tunnel

ip route get 34.141.68.157    # not in list
  -> 34.141.68.157 dev wg0_gnosisvpn src 10.128.0.90            # INTO the tunnel
```

### 4. Measurable cost

With no user applications running, the tunnel interface
(`/sys/class/net/wg0_gnosisvpn/statistics`) still shows a steady **~25–30
packets/sec**, asymmetric (mostly TX), consistent with continuous SYN
retransmissions to unreachable peers. This is a permanent idle floor.

## Root cause

A HOPR node's peer set is **open-ended and discovered at runtime** (gossip/DHT).
The bypass mechanism is a **static, finite list of `/32` routes**. Any peer the
node discovers that is not in that snapshot is routed into the node's own tunnel.
A static allow-list cannot track dynamic peer discovery, so misrouted dials are
inevitable and accumulate over time.

The invariant being violated: **a VPN provider process must never egress through
the tunnel it provides.** Its control-plane traffic (peer dials and the WireGuard
underlay) must always use the underlay interface (eth0). Enforcing this
peer-by-peer via an IP allow-list is the wrong granularity.

## Suggested fix

Scope the bypass to the **process/uid**, not to an enumerated IP list. A
uid-based policy route makes "all traffic owned by the worker uses eth0" a single
invariant, so no future peer can ever be misrouted:

```bash
# Dedicated table that just routes everything out eth0:
ip route add default via 10.138.25.42 dev eth0 table 5182
# Every socket owned by uid 997 (gnosisvpn) consults it before the 0.0.0.0/1 catch-all:
ip rule add uidrange 997-997 lookup 5182 priority 1000
```

The worker's localhost WireGuard underlay (`127.0.0.1:52460`) is in the `local`
table (priority 0) and is unaffected, so the tunnel itself keeps functioning;
this only stops the worker from looping into it.

Equivalent alternatives the HOPR team may prefer:
- An `fwmark`/cgroup-based rule scoped to the worker process instead of uid.
- Binding the worker's outbound peer sockets explicitly to the eth0 address
  (`SO_BINDTODEVICE` / source-IP bind) so they never match the catch-all.

The key point is that the fix should be **process-scoped**, replacing the static
per-IP bypass list, so it is correct for the full, dynamic peer set rather than
for a snapshot of it.

## How to reproduce / inspect

```bash
# Tunnel and eth0 source IPs (tunnel IP changes across reconnects — derive it,
# don't hardcode):
TUN=$(ip -4 -o addr show wg0_gnosisvpn | awk '{print $4}' | cut -d/ -f1)
ETH=$(ip -4 -o addr show eth0          | awk '{print $4}' | cut -d/ -f1)

# All worker peer dials, with process attribution (needs root). The SOURCE IP is
# the classifier: source = tunnel IP -> misrouted/stuck; source = eth0 -> correct.
sudo ss -tnp 'sport = 9091 or dport = 9091'

# Quickest tell — any :9091 dial sourced from the tunnel IP is misrouted:
sudo ss -tnp 'sport = 9091 or dport = 9091' | grep "$TUN"

# Extract the STUCK peer IPs (SYN-SENT, sourced from the tunnel):
sudo ss -tnH state syn-sent 'sport = 9091 or dport = 9091' \
  | awk -v t="$TUN" '$3 ~ "^"t":" {split($4,p,":"); print p[1]}' | sort -u

# Extract the WORKING peer IPs (ESTABLISHED, sourced from eth0):
sudo ss -tnH state established 'sport = 9091 or dport = 9091' \
  | awk -v e="$ETH" '$3 ~ "^"e":" {split($4,p,":"); print p[1]}' | sort -u

# Optional confirmation of WHY each goes where (the routing decision):
ip route get <stuck-peer-ip>     # -> dev wg0_gnosisvpn   (into the tunnel)
ip route get <working-peer-ip>   # -> via 10.138.25.42 dev eth0  (bypasses it)

# Idle traffic floor on the tunnel:
IF=wg0_gnosisvpn
a=$(( $(</sys/class/net/$IF/statistics/rx_packets)+$(</sys/class/net/$IF/statistics/tx_packets) ))
sleep 5
b=$(( $(</sys/class/net/$IF/statistics/rx_packets)+$(</sys/class/net/$IF/statistics/tx_packets) ))
echo "$((b-a)) packets in 5s while idle"
```

Note: `tcpdump` is not installed on this host; the above uses `ss` and the
`/sys/class/net` counters instead.
