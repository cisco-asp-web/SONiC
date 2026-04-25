# Lab 2 Guide: SONiC and FRR – Building a BGP Fabric [60 Min]

We use FRR (Free Range Routing) as the routing stack on SONiC to build a fully functional IP BGP fabric. In this lab we will configure eBGP peering across a leaf-spine topology, verify the BGP control plane, and inspect how routing state is reflected in the SONiC Redis database.

## Contents
- [Lab Objectives](#lab-objectives)
- [Topology](#topology)
- [Introduction to SONiC BGP and FRR](#introduction-to-sonic-bgp-and-frr)
- [BGP Deep Dive](#bgp-deep-dive)
  - [BGP Finite State Machine](#bgp-finite-state-machine)
  - [BGP Message Types](#bgp-message-types)
  - [BGP Path Attributes](#bgp-path-attributes)
  - [BGP Best Path Selection Algorithm](#bgp-best-path-selection-algorithm)
  - [FRR Internal Architecture and IPC](#frr-internal-architecture-and-ipc)
  - [The SONiC BGP Route Pipeline](#the-sonic-bgp-route-pipeline)
  - [Understanding Each FRR Configuration Knob](#understanding-each-frr-configuration-knob)
- [SONiC Interface Configuration and Verification](#sonic-interface-configuration-and-verification)
- [Build a Leaf-Spine BGP Fabric](#build-a-leaf-spine-bgp-fabric)
- [BGP Configuration using SONiC and FRR](#bgp-configuration-using-sonic-and-frr)
- [BGP Verification](#bgp-verification)
- [Redis Database Verification](#redis-database-verification)
- [End of Lab 2](#end-of-lab-2)

## Lab Objectives

We will have achieved the following objectives upon completion of Lab 2:

* Understand how FRR integrates with SONiC as the BGP routing stack
* Understand the `split` routing configuration mode and its file structure
* Configure and verify SONiC interface IP addressing
* Design and implement a two-tier eBGP leaf-spine fabric
* Configure BGP neighbors, address families, and route-maps using `vtysh`
* Verify BGP session state, learned prefixes, and routing table
* Confirm end-to-end reachability across the fabric
* Inspect BGP routing state in the SONiC Redis database


## Topology

For Labs 1–6 you are using a single shared topology. The four SONiC nodes running on Cisco 8000 Emulator form a two-tier fabric with host containers attached to each leaf.

```
                    ┌──────────────────────────────┐
                    │           spine-01            │
                    │            AS 65000           │
                    │      Loopback: 4.4.4.4/32     │
                    │   Loopback: fc00:0:4::1/128   │
                    └────────┬──────────┬───────────┘
                             │          │           │
                   Eth0      │    Eth8  │    Eth16  │
              10.1.1.1/31    │  10.1.1.3/31  10.1.1.5/31
         2001:db8:1:1::1/127 │  2001:db8:1:1::3/127   2001:db8:1:1::5/127
                             │          │           │
              10.1.1.0/31    │  10.1.1.2/31  10.1.1.4/31
         2001:db8:1:1::0/127 │  2001:db8:1:1::2/127   2001:db8:1:1::4/127
                   Eth0      │    Eth0  │    Eth0   │
          ┌──────────────────┘          │           └──────────────────┐
          │                             │                              │
 ┌────────┴──────┐            ┌─────────┴─────┐             ┌─────────┴─────┐
 │    leaf-01    │            │    leaf-02    │             │    leaf-03    │
 │   AS 65001    │            │   AS 65002    │             │   AS 65003    │
 │  1.1.1.1/32   │            │  2.2.2.2/32   │             │  3.3.3.3/32   │
 └──────┬────────┘            └──┬────────┬───┘             └───────┬───────┘
        │  Eth8                  │Eth8    │Eth16                    │Eth8
        │                        │        │                          │
     host-01                  host-01  host-02                   host-03
    (eth1)                    (eth2)   (eth1)                    (eth1)
```

> **Note:** host-01 is dual-homed to both leaf-01 and leaf-02, providing redundant connectivity.

### Management Access

| Device   | Management IP  | SSH Command                 |
|----------|----------------|-----------------------------|
| leaf-01  | 172.20.2.100   | `ssh cisco@172.20.2.100`    |
| leaf-02  | 172.20.2.101   | `ssh cisco@172.20.2.101`    |
| leaf-03  | 172.20.2.102   | `ssh cisco@172.20.2.102`    |
| spine-01 | 172.20.2.103   | `ssh cisco@172.20.2.103`    |


## Introduction to SONiC BGP and FRR

### What is FRR?

FRR (Free Range Routing) is an open-source IP routing protocol suite for Linux. It implements the full range of enterprise routing protocols including BGP, OSPF, IS-IS, PIM, LDP, and more. On SONiC running on Cisco 8000, FRR serves as the routing stack that provides the control plane for all IP routing functions.

FRR consists of a set of daemons, each responsible for a specific protocol:

| Daemon     | Function                                                    |
|------------|-------------------------------------------------------------|
| `zebra`    | Core routing daemon — interfaces with the Linux kernel RIB  |
| `bgpd`     | BGP daemon — manages all BGP sessions and the BGP RIB       |
| `staticd`  | Static route daemon                                         |
| `ospfd`    | OSPF daemon (IPv4)                                          |
| `ospf6d`   | OSPF daemon (IPv6)                                          |
| `isisd`    | IS-IS daemon                                                |

`zebra` acts as the central coordinator. It receives routes from all protocol daemons and programs the best routes into the Linux kernel FIB. SONiC's `fpmsyncd` then reads routes from the kernel and writes them to the Redis APPL_DB, where `orchagent` picks them up and programs the hardware ASIC.

<img src="../drawings/frr-bgp-framework.png" width="800" />

### FRR and SONiC Integration — Split Mode

SONiC supports multiple routing configuration modes. This lab uses **`split` mode**, set by the `docker_routing_config_mode` field in `DEVICE_METADATA`:

```json
"docker_routing_config_mode": "split"
```

In `split` mode:
- FRR runs in its own Docker container called `bgp`
- FRR configuration is stored **separately** from `config_db.json`
- The FRR startup configuration file is: `/etc/sonic/frr/bgpd.conf`
- Changes made via `vtysh` take effect immediately (running config)
- To persist changes across reboots you must save: `copy running-config startup-config`

> **Key Concept:** In SONiC, `config_db.json` owns interface IPs, VLANs, and device metadata. FRR's `bgpd.conf` owns routing protocol configuration. Both files are loaded at boot time into different subsystems.

### The vtysh Shell

`vtysh` is the FRR unified command shell. It provides a single CLI interface across all FRR daemons using IOS-like syntax. You run it as root:

```
cisco@leaf-01:~$ sudo vtysh
```

Once inside vtysh you will see a prompt with the device hostname:

```
leaf-01#
```

Key vtysh commands:

| vtysh Command                    | Description                              |
|----------------------------------|------------------------------------------|
| `show running-config`            | Show the full FRR running configuration  |
| `show bgp summary`               | Show BGP neighbor summary                |
| `show bgp neighbors`             | Show detailed BGP neighbor information   |
| `show bgp ipv4 unicast`          | Show BGP IPv4 RIB                        |
| `show bgp ipv6 unicast`          | Show BGP IPv6 RIB                        |
| `show ip route`                  | Show the full IPv4 routing table         |
| `configure terminal`             | Enter global configuration mode          |
| `copy running-config startup-config` | Save running config to `bgpd.conf`   |
| `write memory`                   | Alias for save                           |

### Verify FRR Container is Running

Before configuring BGP, confirm the FRR BGP container is running on each device:

```
cisco@leaf-01:~$ docker ps | grep bgp
```

Expected output:
```
a3f92b1c8d45   docker-sonic-frr:latest   "/usr/bin/supervisord"   2 hours ago   Up 2 hours   bgp
```

You can also check the FRR daemon status from inside the container:

```
cisco@leaf-01:~$ docker exec -it bgp supervisorctl status
```

Expected output:
```
bgpd                             RUNNING   pid 2847, uptime 2:14:22
zebra                            RUNNING   pid 2846, uptime 2:14:22
staticd                          RUNNING   pid 2848, uptime 2:14:22
fpmsyncd                         RUNNING   pid 2849, uptime 2:14:22
```


## BGP Deep Dive

This section covers the internals of BGP and FRR in detail. Understanding these concepts will make the configuration tasks in this lab intuitive rather than mechanical, and will equip you to diagnose real-world BGP issues.

---

### BGP Finite State Machine

Every BGP session progresses through a well-defined set of states before it becomes operational. This Finite State Machine (FSM) is defined in RFC 4271. FRR implements it exactly. Understanding the FSM is critical for troubleshooting — when a session is stuck in a given state, that state tells you precisely what is failing.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    BGP Finite State Machine                  │
  │                                                              │
  │   [Idle] ──ManualStart──► [Connect] ──TCP OK──► [OpenSent]  │
  │      ▲                        │                     │        │
  │      │                   TCP Fail               OPEN rcvd    │
  │      │                        │                     │        │
  │      │                        ▼                     ▼        │
  │      │                   [Active] ◄──────── [OpenConfirm]    │
  │      │                        │                     │        │
  │   HoldTimer                TCP OK             KEEPALIVE rcvd │
  │   Expire /              [OpenSent]                  │        │
  │   Error                                             ▼        │
  │      │                                        [Established]  │
  │      └────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────────┘
```

#### State Descriptions

**Idle**
The initial state. BGP refuses all incoming connections and makes no outgoing connections. BGP enters Idle when:
- The process just started
- A TCP connection attempt failed (ConnectRetry timer is running)
- A Notification was received or sent
- The hold timer expired

In FRR, a peer stuck in `Idle` is often caused by a routing policy issue (`ebgp-requires-policy`), a missing route to the neighbor, or an administrative shutdown. Check with:
```
cisco@leaf-01:~$ show bgp neighbors 10.1.1.1 | grep "BGP state"
```

**Connect**
BGP is waiting for the TCP three-way handshake to complete. FRR has initiated a TCP SYN to the neighbor's port 179. If the connection succeeds, BGP moves to OpenSent. If it fails, BGP moves to Active.

A peer stuck in `Connect` usually means:
- TCP port 179 is firewalled
- The neighbor IP is unreachable (check the routing table)
- The remote end has no BGP process listening

**Active**
BGP is actively trying to initiate a TCP connection. The distinction between Connect and Active is subtle: in Connect, BGP is waiting on one specific TCP attempt; in Active, BGP is retrying after a failure, backing off with the ConnectRetry timer.

A peer stuck in `Active` usually means the TCP connection keeps failing. Verify with:
```
cisco@leaf-01:~$ sudo tcpdump -i Ethernet0 tcp port 179
```

**OpenSent**
TCP is established. BGP has sent an OPEN message to the peer and is waiting to receive the peer's OPEN message in return. OPEN messages contain: BGP version, local AS number, hold time proposal, and BGP Identifier (router-ID).

If the peer's OPEN is rejected (wrong ASN, incompatible hold time, duplicate router-ID), a NOTIFICATION is sent and the session drops back to Idle.

**OpenConfirm**
Both OPEN messages have been received and validated. BGP has sent a KEEPALIVE to acknowledge the OPEN and is waiting for the peer's KEEPALIVE. Once the peer's KEEPALIVE arrives, the session moves to Established.

**Established**
The BGP session is fully operational. BGP can now exchange UPDATE messages carrying routing information. KEEPALIVE messages are exchanged at the keepalive interval (default 60s in vanilla BGP, but FRR defaults to 3s hold / 1s keepalive for fast convergence in data center designs).

The hold timer is negotiated during OPEN — both peers propose a hold time and the lower value is used. If no KEEPALIVE or UPDATE is received within the hold time, the session is torn down.

#### Session Timers in FRR

| Timer          | Default (FRR) | Purpose                                             |
|----------------|---------------|-----------------------------------------------------|
| Connect Retry  | 120s          | Delay between TCP reconnect attempts                |
| Hold Time      | 9s            | Max time without KEEPALIVE before session tears down |
| Keepalive      | 3s            | Frequency of KEEPALIVE messages (1/3 of hold time) |
| Advertisement  | 0s (eBGP)     | Min time between UPDATE messages to a peer          |

> **FRR vs traditional BGP:** Standard BGP (RFC 4271) uses 90s keepalive / 270s hold time. FRR's defaults (3s/9s) are tuned for data center environments where fast failure detection matters more than message overhead.

---

### BGP Message Types

BGP uses four message types, all carried over TCP port 179. Every message starts with a 19-byte common header: 16 bytes of marker (all 0xFF, for legacy authentication detection), 2 bytes of length, and 1 byte of type.

#### OPEN (Type 1)

Sent immediately after TCP is established. Contains:

| Field              | Size     | Description                                              |
|--------------------|----------|----------------------------------------------------------|
| Version            | 1 byte   | Always 4 (BGP-4)                                         |
| My Autonomous System | 2 bytes | Local AS number (or AS 23456 for 4-byte AS via capability) |
| Hold Time          | 2 bytes  | Proposed hold timer in seconds                           |
| BGP Identifier     | 4 bytes  | Router-ID (must be unique in the AS)                    |
| Optional Parameters | variable | BGP capabilities advertised (4-byte AS, AddPath, etc.)  |

Key capabilities negotiated in OPEN (visible in `show bgp neighbors`):
- **4-Byte AS Number** (RFC 6793) — allows ASNs above 65535
- **Route Refresh** (RFC 2918) — allows requesting a fresh BGP table without session reset
- **Multiprotocol Extensions** (RFC 4760) — enables IPv6 (AFI=2, SAFI=1) and other address families
- **AddPath** (RFC 7911) — allows advertising multiple paths for the same prefix

#### UPDATE (Type 2)

The most complex message type. Carries route advertisements and withdrawals:

```
UPDATE message structure:
┌─────────────────────────────────────┐
│  Unfeasible Routes Length (2 bytes) │  ← length of withdrawn routes
├─────────────────────────────────────┤
│  Withdrawn Routes (variable)        │  ← list of prefixes to withdraw
├─────────────────────────────────────┤
│  Total Path Attribute Length        │  ← length of path attributes
├─────────────────────────────────────┤
│  Path Attributes (variable)         │  ← AS_PATH, NEXT_HOP, MED, etc.
├─────────────────────────────────────┤
│  NLRI (variable)                    │  ← prefixes being advertised
└─────────────────────────────────────┘
```

A single UPDATE can carry:
- Withdrawals only (NLRI is empty)
- New routes only (withdrawn routes length is 0)
- Both withdrawals and new routes simultaneously

In FRR, you can trigger an UPDATE refresh with:
```
cisco@leaf-01:~$ sudo vtysh -c "clear bgp 10.1.1.1 soft"
```
This requests a Route Refresh (soft reset) — the peer resends its full table without tearing down the session.

#### KEEPALIVE (Type 3)

A minimal 19-byte message (just the header). Sent to confirm the session is alive within the hold time. No payload — the presence of the message is the signal.

FRR sends a KEEPALIVE in response to an OPEN (to move to Established) and then periodically at the keepalive interval.

#### NOTIFICATION (Type 4)

Sent when an error is detected. Immediately closes the TCP connection after sending. Contains:
- Error Code (1 byte)
- Error Subcode (1 byte)
- Data (variable — the offending message or field)

Common Error Codes and their meaning:

| Code | Subcode | Meaning                             | Common Cause                          |
|------|---------|-------------------------------------|---------------------------------------|
| 1    | 2       | Unsupported version number          | Peer running BGP-3 (extremely rare)   |
| 1    | 2       | Bad peer AS                         | `remote-as` mismatch                  |
| 1    | 3       | Bad BGP Identifier                  | Duplicate router-ID in iBGP          |
| 2    | 2       | Unrecognized attribute              | Attribute type not understood         |
| 2    | 3       | Missing well-known attribute        | AS_PATH or NEXT_HOP absent            |
| 3    | 1       | Hold Timer Expired                  | Missed 3 consecutive KALIVEs         |
| 6    | 2       | Administrative shutdown             | `neighbor X shutdown` configured      |
| 6    | 3       | Peer de-configured                  | Neighbor removed from config          |

In FRR, NOTIFICATION events are logged. Check the BGP log:
```
cisco@leaf-01:~$ docker exec bgp tail -f /var/log/frr/bgpd.log
```

---

### BGP Path Attributes

Path attributes are carried in UPDATE messages and describe the properties of the advertised prefixes. They are the mechanism through which BGP makes routing decisions and implements policy.

#### Attribute Categories

BGP attributes are classified along two axes:

**Well-known vs Optional:**
- **Well-known**: Every BGP implementation must recognize these attributes
- **Optional**: Implementations may not understand them (and must handle them gracefully)

**Mandatory vs Discretionary (for well-known) / Transitive vs Non-transitive (for optional):**
- **Mandatory**: Must be present in every UPDATE that advertises a route
- **Discretionary**: May be omitted
- **Transitive**: If an optional attribute is not recognized, it is still forwarded to other peers
- **Non-transitive**: If not recognized, the attribute is silently dropped

#### Key Attributes Used in This Lab

**ORIGIN (Type 1)** — Well-known Mandatory
Indicates how the prefix entered BGP:
- `i` (IGP) — injected via `network` statement (used in this lab)
- `e` (EGP) — learned from the old EGP protocol (legacy, not used)
- `?` (Incomplete) — redistributed from another protocol (e.g., OSPF → BGP)

In FRR output: `Origin IGP` = `i`. The `network` command in this lab sets Origin to IGP.

**AS_PATH (Type 2)** — Well-known Mandatory
An ordered list of AS numbers that the route has traversed. This is BGP's primary loop prevention mechanism.

When a BGP router receives an UPDATE:
1. It checks if its own AS number appears in the AS_PATH
2. If yes → loop detected → route is silently discarded
3. If no → route is accepted, and the local AS is prepended before the route is forwarded

In this lab, when spine-01 (AS 65000) receives `1.1.1.1/32` from leaf-01 (AS 65001), it sees `AS_PATH = [65001]`. When it advertises this to leaf-02 (AS 65002), it prepends its own ASN: `AS_PATH = [65000, 65001]`. Leaf-02 accepts it because neither 65002 nor any loop exists in the path.

**NEXT_HOP (Type 3)** — Well-known Mandatory
The IP address of the next-hop router for this prefix.

For eBGP: the NEXT_HOP is always set to the IP address of the advertising router's interface on the link between the two peers. Leaf-01 (10.1.1.0) advertises `1.1.1.1/32` to spine-01 with NEXT_HOP = 10.1.1.0. The spine then advertises this to leaf-02 with NEXT_HOP = 10.1.1.1 (its own address toward leaf-02's link — because eBGP changes the NEXT_HOP at each hop).

> **iBGP vs eBGP NEXT_HOP behavior:** In iBGP, the NEXT_HOP is NOT changed. This is why iBGP requires either a full mesh or a Route Reflector — every router must have a route to the NEXT_HOP. In eBGP (used in this lab), the NEXT_HOP is always updated.

**MULTI_EXIT_DISC / MED (Type 4)** — Optional Non-transitive
A hint to external neighbors about the preferred entry point into an AS when there are multiple links. Lower MED is preferred. Only compared between routes from the same neighboring AS. Not propagated beyond the next AS.

**LOCAL_PREF (Type 5)** — Well-known Discretionary (iBGP only)
Used within an AS to indicate the preferred exit path. Higher LOCAL_PREF is preferred. Not sent to eBGP peers. Since this lab uses only eBGP, LOCAL_PREF is not relevant here — but FRR will set it to the default value of 100 on all received routes.

**COMMUNITY (Type 8)** — Optional Transitive
32-bit values attached to routes to carry policy tags. Commonly used for route filtering and traffic engineering. Format: `ASN:value` (e.g., `65000:100`). Standard communities include:
- `no-export` (0xFFFFFF01): Do not advertise outside the local AS
- `no-advertise` (0xFFFFFF02): Do not advertise to any peer

**ATOMIC_AGGREGATE (Type 6) and AGGREGATOR (Type 7)**
Used when route aggregation is performed. Not relevant in this lab.

#### Attribute Summary Table

| Type | Name           | Category                    | Used in this lab |
|------|----------------|-----------------------------|-----------------|
| 1    | ORIGIN         | Well-known Mandatory        | Yes (IGP)       |
| 2    | AS_PATH        | Well-known Mandatory        | Yes             |
| 3    | NEXT_HOP       | Well-known Mandatory        | Yes             |
| 4    | MED            | Optional Non-transitive     | No              |
| 5    | LOCAL_PREF     | Well-known Discretionary    | Implicit (100)  |
| 6    | ATOMIC_AGGR    | Well-known Discretionary    | No              |
| 7    | AGGREGATOR     | Optional Transitive         | No              |
| 8    | COMMUNITY      | Optional Transitive         | No              |
| 9    | ORIGINATOR_ID  | Optional Non-transitive     | No (iBGP RR)    |
| 10   | CLUSTER_LIST   | Optional Non-transitive     | No (iBGP RR)    |
| 14   | MP_REACH_NLRI  | Optional Non-transitive     | Yes (IPv6)      |
| 15   | MP_UNREACH_NLRI| Optional Non-transitive     | Yes (IPv6)      |

> **MP_REACH_NLRI and MP_UNREACH_NLRI** are how IPv6 (and other address families) are carried in BGP UPDATE messages. The base BGP UPDATE format only has fields for IPv4 NLRI — all other address families are carried in these extension attributes. This is why you must explicitly activate IPv6 with `address-family ipv6 unicast` in FRR.

---

### BGP Best Path Selection Algorithm

When BGP receives multiple paths to the same prefix from different peers, it must select exactly one **best path** to install in the routing table (and one to advertise to peers). FRR implements the standard BGP decision process in the order below. The first criterion that produces a winner stops the evaluation.

This is evaluated **per-prefix** every time a new path is received or an existing path is withdrawn.

```
Step  1: Is the NEXT_HOP reachable?         → Unreachable paths are invalid, skip them
Step  2: Prefer highest WEIGHT              → FRR/Cisco proprietary, local significance only
Step  3: Prefer highest LOCAL_PREF         → iBGP only; all eBGP paths get LOCAL_PREF 100
Step  4: Prefer locally originated routes  → network/aggregate/redistribute > eBGP > iBGP
Step  5: Prefer shortest AS_PATH           → Fewer AS hops = better
Step  6: Prefer lowest ORIGIN code         → IGP (i) < EGP (e) < Incomplete (?)
Step  7: Prefer lowest MED                 → Only compared if paths are from the same AS
Step  8: Prefer eBGP over iBGP            → External routes preferred over internal
Step  9: Prefer lowest IGP metric to NH   → Prefer the path whose next-hop is closest
Step 10: Prefer oldest eBGP path          → Stability: avoid flapping (if enabled)
Step 11: Prefer lowest BGP Router-ID      → Tiebreak: prefer peer with lowest router-ID
Step 12: Prefer shortest Cluster List     → iBGP Route Reflector tiebreak
Step 13: Prefer lowest neighbor address   → Final tiebreak: lowest peer IP wins
```

#### How This Applies to Our Lab

In our fabric, leaf-01 receives only **one path** to 2.2.2.2/32 (via spine-01). No tie-breaking is needed. But on spine-01, consider what happens as more prefixes are added in later labs — the path selection becomes critical.

**Key step for our fabric: Step 5 (AS_PATH length)**

Leaf-01 seeing `2.2.2.2/32`:
- Path via spine-01: AS_PATH = `[65000, 65002]` — length 2

There is only one path, so it wins. But in a dual-spine design (future lab), both spines would advertise the same prefix with equal AS_PATH length. The decision would fall to Step 13 (lowest neighbor IP), which is deterministic but may not match your traffic engineering intent. This is where `bgp bestpath as-path multipath-relax` and `maximum-paths` become essential.

#### ECMP and `bgp bestpath as-path multipath-relax`

By default, BGP only installs multiple paths when they have **identical** AS_PATH. In our fabric, if spine-01 and a hypothetical spine-02 both advertise `2.2.2.2/32`, the paths look like:
- Via spine-01 (AS 65000): AS_PATH = `[65000, 65002]`
- Via spine-02 (AS 65100): AS_PATH = `[65100, 65002]`

These AS_PATHs are different — so without the multipath-relax knob, only one path would be installed and load-balancing would not occur.

`bgp bestpath as-path multipath-relax` tells FRR: "ignore AS_PATH differences when evaluating multipath eligibility — only require equal AS_PATH **length**." With this knob, both spine paths above are ECMP candidates (length 2 = length 2).

`maximum-paths 64` then allows up to 64 equal-cost paths to be installed simultaneously in the FIB, enabling hardware-level ECMP hashing across all equal-cost next-hops.

---

### FRR Internal Architecture and IPC

Understanding how FRR daemons communicate internally clarifies what actually happens when you type a `vtysh` command.

#### The ZAPI Protocol

FRR daemons communicate using **ZAPI** (Zebra API), a private binary protocol over Unix domain sockets. Each protocol daemon (bgpd, ospfd, etc.) maintains a persistent ZAPI connection to zebra.

```
┌─────────────────────────────────────────────────────────┐
│                    FRR Process Model                    │
│                                                         │
│  ┌─────────┐    ZAPI socket    ┌────────┐              │
│  │  bgpd   │◄──────────────────►│        │              │
│  └─────────┘                   │        │  Netlink      │
│                                │ zebra  │◄─────────────►│ Linux Kernel │
│  ┌─────────┐    ZAPI socket    │        │              │
│  │ staticd │◄──────────────────►│        │              │
│  └─────────┘                   └────────┘              │
│                                                         │
│  ┌─────────┐    Unix socket                             │
│  │  vtysh  │◄────────────────── to each daemon          │
│  └─────────┘                                            │
└─────────────────────────────────────────────────────────┘
```

**What bgpd sends to zebra via ZAPI:**
- `ZEBRA_ROUTE_ADD` — install a BGP-learned route into the kernel
- `ZEBRA_ROUTE_DELETE` — remove a route
- `ZEBRA_REDISTRIBUTE_ADD` — request redistribution of connected/static routes into BGP
- `ZEBRA_NEXTHOP_REGISTER` — register a next-hop for tracking (NHT)

**What zebra sends to bgpd via ZAPI:**
- `ZEBRA_NEXTHOP_UPDATE` — a tracked next-hop's reachability changed
- `ZEBRA_REDISTRIBUTE_ROUTE_ADD` — a redistributed route is available

#### vtysh Architecture

`vtysh` is not a daemon — it is a **client program** that connects to each running FRR daemon over per-daemon Unix sockets (typically in `/var/run/frr/`). When you type `show bgp summary` in vtysh:

1. vtysh sends the command string to `bgpd`'s Unix socket
2. `bgpd` processes the command, formats the output
3. `bgpd` sends the formatted text back to vtysh
4. vtysh prints it to your terminal

When you type `show ip route` in vtysh, vtysh sends it to `zebra` instead. When a command affects multiple daemons (like `show running-config`), vtysh fans the command out to all daemons and concatenates the results.

This is why `vtysh` requires `sudo` — it needs permission to access the privileged Unix sockets owned by the FRR daemons.

#### How a BGP Route Gets Installed

Here is the exact sequence of events when leaf-01 learns `2.2.2.2/32` from spine-01:

```
1. bgpd receives TCP segment on port 179 from spine-01
2. bgpd parses the BGP UPDATE message
3. bgpd extracts the NLRI (2.2.2.2/32) and path attributes
4. bgpd runs the Best Path Selection Algorithm
5. 2.2.2.2/32 is selected as best path (only path)
6. bgpd sends ZEBRA_ROUTE_ADD to zebra via ZAPI
   → prefix: 2.2.2.2/32, nexthop: 10.1.1.1, ifindex: Ethernet0
7. zebra validates the next-hop (10.1.1.1 is reachable via Ethernet0)
8. zebra calls rtnetlink (Netlink) to install the route in the Linux kernel
   → ip route add 2.2.2.2/32 via 10.1.1.1 dev Ethernet0 proto bgp
9. fpmsyncd (running inside the bgp container) reads the Netlink route event
10. fpmsyncd writes to Redis APPL_DB: ROUTE_TABLE:2.2.2.2/32
11. orchagent (in the swss container) reads from APPL_DB
12. orchagent translates the route to SAI (Switch Abstraction Interface) calls
13. SAI driver programs the route into the hardware ASIC FIB
```

You can observe this pipeline step by step:
```bash
# Step 6-8: Verify the route in the Linux kernel
cisco@leaf-01:~$ ip route show 2.2.2.2/32

# Step 10: Verify in Redis APPL_DB
cisco@leaf-01:~$ redis-cli -n 0 hgetall "ROUTE_TABLE:2.2.2.2/32"

# Step 12-13: Verify in ASIC_DB (the SAI route object)
cisco@leaf-01:~$ redis-cli -n 1 keys "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:*2.2.2.2*"
```

---

### The SONiC BGP Route Pipeline

The complete route programming pipeline in SONiC involves multiple containers and processes:

```
┌──────────────────────────────────────────────────────────────────┐
│                      SONiC BGP Container                        │
│                                                                  │
│   ┌────────┐  ZAPI  ┌────────┐  Netlink  ┌──────────────────┐  │
│   │  bgpd  │◄──────►│ zebra  │──────────►│  Linux Kernel    │  │
│   └────────┘        └────────┘           │  Route Table     │  │
│                                          └────────┬─────────┘  │
│                     ┌──────────┐                  │            │
│                     │fpmsyncd  │◄─── Netlink ──────┘            │
│                     └────┬─────┘  (route events)               │
└──────────────────────────┼───────────────────────────────────────┘
                           │ Redis APPL_DB
                           │ ROUTE_TABLE write
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                      SONiC SWSS Container                       │
│                                                                  │
│   ┌──────────┐  reads  ┌──────────────────────────────────┐    │
│   │orchagent │◄────────│           Redis                  │    │
│   └────┬─────┘  APPL_DB│  APPL_DB(0) │ ASIC_DB(1) │ ...  │    │
│        │               └─────────────┼────────────┘       │    │
│        │ SAI calls             writes │                    │    │
│        ▼                             ▼                     │    │
│   ┌──────────┐                ┌───────────┐               │    │
│   │   SAI    │                │  ASIC_DB  │               │    │
│   │ (vendor) │                │ (Redis 1) │               │    │
│   └────┬─────┘                └───────────┘               │    │
└────────┼─────────────────────────────────────────────────────────┘
         │ hardware programming
         ▼
┌──────────────────────────────────────────────────────────────────┐
│              Cisco 8000 ASIC (Silicon One)                      │
│                    Hardware FIB                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Key containers and their roles:**

| Container | Process     | Role                                                        |
|-----------|-------------|-------------------------------------------------------------|
| `bgp`     | bgpd        | BGP protocol daemon — manages sessions and BGP RIB         |
| `bgp`     | zebra       | FRR core — RIB manager, nexthop resolution, Netlink writes  |
| `bgp`     | fpmsyncd    | Reads kernel routes via Netlink, writes to APPL_DB         |
| `swss`    | orchagent   | Reads APPL_DB, translates to SAI calls, writes to ASIC_DB  |
| `syncd`   | syncd       | Reads ASIC_DB, calls vendor SAI library to program hardware |

**Why the indirection through Redis?**

SONiC uses Redis as a decoupled message bus between containers so that:
- Each container can be restarted independently without losing state
- Multiple consumers can subscribe to the same data (e.g., both orchagent and a monitoring agent can watch APPL_DB)
- State is persisted across container restarts
- The architecture is vendor-agnostic — only the SAI layer differs between hardware platforms

---

### Understanding Each FRR Configuration Knob

Every line of BGP configuration in this lab has a specific purpose. This section explains each one in depth.

#### `bgp router-id 1.1.1.1`

The BGP Router-ID is a 32-bit identifier (displayed as a dotted-quad) that uniquely identifies this BGP speaker within the network. It is carried in the BGP OPEN message.

**Rules:**
- Must be globally unique within the BGP domain (not just the AS)
- If two iBGP speakers have the same router-ID, sessions fail with NOTIFICATION error code 1, subcode 3 (Bad BGP Identifier)
- FRR will automatically select the highest loopback IP as the router-ID if not configured explicitly — but explicit configuration is a best practice

**Why use the loopback IP?** Using the Loopback0 address (e.g., 1.1.1.1) as the router-ID is conventional because it is stable — it doesn't change if a physical interface goes down.

#### `bgp log-neighbor-changes`

Tells FRR to log a message to syslog whenever a BGP neighbor transitions state (Up, Down, Reset). Without this, session flaps may go undetected. Example log entry:

```
%BGP-5-ADJCHANGE: neighbor 10.1.1.1 Up
%BGP-5-ADJCHANGE: neighbor 10.1.1.1 Down (BGP Notification received)
```

View these events:
```
cisco@leaf-01:~$ docker exec bgp cat /var/log/frr/bgpd.log | grep ADJCHANGE
```

#### `no bgp ebgp-requires-policy`

**Default behavior (without this command):** FRR requires that every eBGP neighbor has both an inbound and an outbound route-map configured before it will exchange routes. If any policy is missing, no routes are exchanged. This is a security-hardening feature introduced in FRR 5.0+ to prevent accidental full routing table leaks.

**With `no bgp ebgp-requires-policy`:** This check is disabled. Any routes that match the address-family activation (`activate`) will be exchanged without requiring explicit policy.

In a production deployment, you would configure explicit import/export policies. In this lab, we disable the requirement for simplicity so we can focus on the BGP fundamentals.

> **Security Note:** In production, always configure inbound policies to filter unexpected prefixes and outbound policies to control what you advertise. Without these, a misconfigured peer could inject arbitrary routes.

#### `no bgp default ipv4-unicast`

**Default behavior (without this command):** FRR automatically activates every neighbor in the IPv4 unicast address family. Any neighbor you define immediately starts exchanging IPv4 routes.

**With `no bgp default ipv4-unicast`:** Neighbors are not activated in any address family by default. You must explicitly run `neighbor X activate` inside each `address-family` block.

**Why this is preferred:** Explicit activation makes the configuration self-documenting and prevents accidentally running IPv4 on a neighbor you intended to configure for IPv6-only. It also matches modern data center BGP design practices.

#### `bgp bestpath as-path multipath-relax`

Standard RFC BGP ECMP requires that multiple paths to the same prefix have **identical AS_PATHs** to qualify as equal-cost multipath. In a dual-spine design, paths through spine-01 (AS 65000) and spine-02 (AS 65100) would have different AS_PATHs — so no ECMP.

This knob relaxes the requirement: paths are ECMP-eligible if they have the same AS_PATH **length**, regardless of the actual AS numbers in the path.

**Effect in this lab:** With one spine, this knob has no immediate effect — there is only one path per prefix. But it is included as a best practice for when the topology expands. Removing it in a dual-spine topology would silently break ECMP without any error message.

#### `no bgp network import-check`

**Default behavior:** When you use `network X.X.X.X/mask` to advertise a prefix into BGP, FRR checks that the exact prefix is present in the routing table before advertising it. If the prefix is not in the RIB (e.g., the loopback interface is down), the advertisement is suppressed.

**With `no bgp network import-check`:** The RIB check is skipped. The prefix is advertised regardless of whether it exists in the routing table.

In this lab, our loopback addresses are always present (loopback interfaces never go down), so this knob doesn't change behavior. However, it prevents a subtle issue: if SONiC's loopback address is briefly removed from the RIB during a configuration reload, BGP would otherwise withdraw and re-advertise the loopback prefix, causing unnecessary churn.

#### `neighbor X remote-as Y`

Defines a BGP neighbor relationship. FRR will:
1. Attempt to open a TCP connection to IP address X on port 179
2. Send a BGP OPEN with local AS number
3. Verify that the peer's OPEN contains AS number Y (if not, send NOTIFICATION and drop)

For eBGP, the neighbor IP must be directly reachable (a connected route). FRR enforces this by default for eBGP — if the peer IP is not reachable via a connected route, the session is rejected.

> **eBGP Multihop:** If you need eBGP sessions across non-directly-connected links, you must configure `neighbor X ebgp-multihop N`. Not used in this lab.

#### `address-family ipv4 unicast` / `address-family ipv6 unicast`

BGP is a multi-protocol routing protocol. Each address family (IPv4 unicast, IPv6 unicast, VPNv4, EVPN, etc.) is managed independently within the same BGP session. This separation allows you to activate different families on different neighbors and apply independent policies.

The address family block contains:
- `network` — prefixes to originate into BGP
- `neighbor X activate` — enable this address family for neighbor X
- `neighbor X route-map NAME in/out` — apply policy
- `maximum-paths N` — ECMP configuration

Only neighbors that are explicitly `activate`d in an address family participate in that family's route exchange.

#### `network 1.1.1.1/32`

Originates the prefix `1.1.1.1/32` into the BGP RIB as a locally generated route. Key behaviors:
- Sets ORIGIN = IGP (code `i`)
- Sets AS_PATH = empty (the local AS will be prepended when advertised to eBGP peers)
- Sets NEXT_HOP = 0.0.0.0 (FRR's way of indicating a local route — changed to the interface IP when advertised)
- The prefix must exist in the RIB unless `no bgp network import-check` is set

This is different from **redistribution**: `network` originates a specific prefix explicitly. Redistribution (`redistribute connected`) would inject all connected routes, which is less precise and can cause unintended prefix leaks.

#### `maximum-paths 64`

Allows FRR to install up to 64 equal-cost BGP paths in the FIB simultaneously. Without this, only one best path is installed. ECMP requires:
1. Multiple paths to the same prefix pass the best-path algorithm with equal preference
2. `maximum-paths N` allows N paths to be installed
3. The ASIC performs per-flow (5-tuple hash) load balancing across all installed next-hops

The Cisco 8000's Silicon One ASIC supports hardware ECMP hashing. The maximum value of 64 ensures we never artificially limit ECMP in the hardware.

#### `route-map RM_SET_SRC permit 10`

Route maps are ordered lists of `permit` or `deny` clauses, each with a sequence number. They are evaluated top-to-bottom; the first matching clause wins. An implicit `deny all` exists at the end of every route map.

This particular route map:
- Sequence 10 — the only clause
- `permit` — if matched, apply the `set` actions and allow the route through
- No `match` conditions — matches ALL routes (catch-all)
- `set src 1.1.1.1` — sets the preferred source address for this route

The route map is applied to the **protocol** (not a neighbor):
```
ip protocol bgp route-map RM_SET_SRC
```

This instructs the Linux kernel: for any BGP-installed route, use `1.1.1.1` as the source address when originating traffic toward that route's next-hop. This affects locally-sourced traffic (like ping), not transit traffic.

**Why is this needed?** Without `RM_SET_SRC`, when leaf-01 pings `2.2.2.2`, Linux would source the packet from `10.1.1.0` (the outgoing interface IP). Leaf-02 would receive an ICMP request from `10.1.1.0`, but the reply would be sent to `10.1.1.0` which is only reachable via the spine — creating an asymmetric path that may not work. By sourcing from the loopback (`1.1.1.1`), the return path is also BGP-routed and fully symmetric.

#### `route-map BGP-IPV6 permit 20` / `set ipv6 next-hop prefer-global`

IPv6 interfaces have both a **link-local** address (fe80::/10, automatically generated from the MAC) and a **global** address (in our case, 2001:db8:1:1::x/127).

By default, FRR may use the link-local address as the IPv6 NEXT_HOP in BGP UPDATEs. Link-local addresses are only valid on the local segment and cannot be routed — using them as next-hops for BGP-learned prefixes would create a broken routing table.

`set ipv6 next-hop prefer-global` overrides this behavior and forces FRR to use the global IPv6 address as the NEXT_HOP in the BGP RIB. This is applied **inbound** on each IPv6 neighbor so that when we receive routes from a peer, the next-hop stored in our BGP RIB is the peer's global address (e.g., `2001:db8:1:1::1`), not its link-local address.

#### `ip nht resolve-via-default` / `ipv6 nht resolve-via-default`

**NHT = Next-Hop Tracking.** FRR's zebra monitors the reachability of BGP next-hops. When a next-hop becomes unreachable, zebra notifies bgpd via ZAPI (`ZEBRA_NEXTHOP_UPDATE`), and bgpd marks all routes through that next-hop as invalid.

By default, NHT uses only specific (non-default) routes to validate next-hop reachability. `resolve-via-default` allows the default route (`0.0.0.0/0` or `::/0`) to be used for next-hop resolution. Without this, if the next-hop is only reachable via a default route, NHT would consider it invalid and suppress the BGP session.

In this lab, since next-hops are directly connected (/31 links), this knob doesn't change behavior — but it is included as a best practice for topologies where next-hops might be reachable only via a default route.

---

## SONiC Interface Configuration and Verification

Before configuring BGP we need to confirm that all interfaces have the correct IP addresses assigned from Lab 1. BGP sessions are established using these point-to-point IP addresses as neighbors.

### Interface IP Address Plan

| Device   | Interface  | IPv4 Address   | IPv6 Address            |
|----------|------------|----------------|-------------------------|
| leaf-01  | Loopback0  | 1.1.1.1/32     | fc00:0:1::1/128         |
| leaf-01  | Ethernet0  | 10.1.1.0/31    | 2001:db8:1:1::0/127     |
| leaf-02  | Loopback0  | 2.2.2.2/32     | fc00:0:2::1/128         |
| leaf-02  | Ethernet0  | 10.1.1.2/31    | 2001:db8:1:1::2/127     |
| leaf-03  | Loopback0  | 3.3.3.3/32     | fc00:0:3::1/128         |
| leaf-03  | Ethernet0  | 10.1.1.4/31    | 2001:db8:1:1::4/127     |
| spine-01 | Loopback0  | 4.4.4.4/32     | fc00:0:4::1/128         |
| spine-01 | Ethernet0  | 10.1.1.1/31    | 2001:db8:1:1::1/127     |
| spine-01 | Ethernet8  | 10.1.1.3/31    | 2001:db8:1:1::3/127     |
| spine-01 | Ethernet16 | 10.1.1.5/31    | 2001:db8:1:1::5/127     |

### Verify Interface Status

On **leaf-01**, confirm all fabric-facing interfaces are operationally up:

```
cisco@leaf-01:~$ show interface status
```

Expected output (relevant interfaces shown):
```
  Interface            Lanes    Speed    MTU    FEC      Alias    Vlan    Oper    Admin    Type    Asym PFC
-----------  ---------------  -------  -----  -----  -------  ------  ------  -------  ------  ----------
  Ethernet0  2304,2305,...    400G     9100   none     etp0    routed    up       up    QSFP-DD      N/A
  Ethernet8  2320,2321,...    400G     9100   none     etp1    routed    up       up    QSFP-DD      N/A
```

### Verify IP Addresses

Confirm the IPv4 and IPv6 addresses are correctly assigned:

```
cisco@leaf-01:~$ show ip interfaces
```

Expected output:
```
Interface       Master    IPv4 address/mask    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -------------------  ------------  --------------  -------------
Ethernet0                  10.1.1.0/31           up/up         N/A             N/A
Loopback0                  1.1.1.1/32            up/up         N/A             N/A
docker0                    240.127.1.1/24        up/up         N/A             N/A
eth0                       172.20.2.100/24       up/up         N/A             N/A
lo                         127.0.0.1/8           up/up         N/A             N/A
```

Check IPv6 addresses:

```
cisco@leaf-01:~$ show ipv6 interfaces
```

Expected output:
```
Interface       Master    IPv6 address/mask                    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -----------------------------------  ------------  --------------  -------------
Ethernet0              2001:db8:1:1::0/127                      up/up        N/A              N/A
Loopback0              fc00:0:1::1/128                          up/up        N/A              N/A
```

### Verify Point-to-Point Connectivity

Before proceeding with BGP, verify Layer 3 reachability to the spine across each link:

```
cisco@leaf-01:~$ ping 10.1.1.1 -c 3
```

Expected output:
```
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=1.24 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.98 ms
64 bytes from 10.1.1.1: icmp_seq=3 ttl=64 time=1.01 ms

--- 10.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.980/1.076/1.240/0.118 ms
```

Verify IPv6 reachability:

```
cisco@leaf-01:~$ ping6 2001:db8:1:1::1 -c 3
```

> **Troubleshooting:** If ping fails, go back and verify the interface configuration in `config_db.json` and confirm `Ethernet0` has `admin_status: up` and the correct IP addresses assigned.


## Build a Leaf-Spine BGP Fabric

### BGP Design Principles

This lab implements a standard two-tier IP fabric using eBGP (External BGP). Key design decisions:

**Why eBGP for a fabric?**
eBGP is widely used in modern data center fabrics (as defined in RFC 7938) because:
- Simple, loop-free routing with AS-PATH as the loop prevention mechanism
- No IGP needed — BGP carries all fabric routes
- Scales well as the fabric grows
- Fine-grained traffic engineering per prefix

**AS Number Assignment:**

| Device   | ASN   | Role  |
|----------|-------|-------|
| spine-01 | 65000 | Spine |
| leaf-01  | 65001 | Leaf  |
| leaf-02  | 65002 | Leaf  |
| leaf-03  | 65003 | Leaf  |

Each leaf uses a **unique private ASN**. This is important: if all leaves shared the same ASN, the spine would see the AS-PATH loop and reject routes propagated between leaves.

**What gets advertised:**
- Each node advertises its Loopback0 prefix into BGP
- Loopback addresses serve as stable, router-ID anchors and are used for management, services, and as BGP next-hops in advanced designs

**BGP Feature Flags Used:**

| FRR Knob                          | Purpose                                                       |
|-----------------------------------|---------------------------------------------------------------|
| `no bgp ebgp-requires-policy`     | Disables mandatory import/export policy check (lab simplification) |
| `no bgp default ipv4-unicast`     | Requires explicit per-neighbor address-family activation      |
| `bgp bestpath as-path multipath-relax` | Allows ECMP across neighbors with different AS-PATHs     |
| `no bgp network import-check`     | Allows advertising prefixes not present in RIB               |
| `maximum-paths 64`                | Enable up to 64 ECMP paths per prefix                        |


### Route Maps Explained

Three route maps are used in this lab:

**`RM_SET_SRC`** — Sets the source address for BGP-learned IPv4 routes in the kernel. Without this, Linux would source traffic from whatever interface is chosen by the kernel, which can cause asymmetric routing.

**`RM_SET_SRC6`** — Same function for IPv6 BGP-learned routes.

**`BGP-IPV6`** — Applied inbound on IPv6 neighbors. Sets the next-hop to prefer the global IPv6 address over the link-local address. This ensures correct forwarding behavior for IPv6 routes learned over link-local sessions.


## BGP Configuration using SONiC and FRR

### Configuration Methods

There are two ways to configure FRR on SONiC in `split` mode:

**Method 1 — Interactive via `vtysh`** (used in this lab)
Configure BGP directly in the FRR shell. Changes take effect immediately. You must save with `copy running-config startup-config` to persist across reboots.

**Method 2 — Edit `bgpd.conf` directly**
Edit `/etc/sonic/frr/bgpd.conf` and restart the BGP container:
```
sudo systemctl restart bgp
```

In this lab we use **Method 1** to build familiarity with the FRR CLI.

### Configuration Modes in vtysh

vtysh uses a hierarchical configuration mode system. Understanding which mode you are in is essential:

```
leaf-01#                          ← EXEC mode (show commands only)
leaf-01# configure terminal
leaf-01(config)#                  ← Global CONFIG mode
leaf-01(config)# router bgp 65001
leaf-01(config-router)#           ← BGP ROUTER mode
leaf-01(config-router)# address-family ipv4 unicast
leaf-01(config-router-af)#        ← ADDRESS FAMILY mode
leaf-01(config-router-af)# exit-address-family
leaf-01(config-router)#           ← Back to BGP ROUTER mode
leaf-01(config-router)# exit
leaf-01(config)#                  ← Back to Global CONFIG mode
leaf-01(config)# end              ← Return to EXEC mode (shortcut from any depth)
leaf-01#
```

You can run `show` commands from EXEC mode at any time without exiting configuration mode by prefixing with `do`:
```
leaf-01(config-router)# do show bgp summary
```

---

### Task 1: Configure BGP on spine-01

spine-01 peers with all three leaves. It is the hub of our eBGP fabric — every inter-leaf route transits through it.

SSH to spine-01:
```
ssh cisco@172.20.2.103
```

Enter the FRR shell:
```
cisco@spine-01:~$ sudo vtysh
```

Enter global configuration mode:
```
spine-01# configure terminal
```

Create the BGP process and set the router-id:
```
spine-01(config)# router bgp 65000
spine-01(config-router)# bgp router-id 4.4.4.4
spine-01(config-router)# bgp log-neighbor-changes
spine-01(config-router)# no bgp ebgp-requires-policy
spine-01(config-router)# no bgp default ipv4-unicast
spine-01(config-router)# bgp bestpath as-path multipath-relax
spine-01(config-router)# no bgp network import-check
```

Define the three leaf neighbors (IPv4):
```
spine-01(config-router)# neighbor 10.1.1.0 remote-as 65001
spine-01(config-router)# neighbor 10.1.1.2 remote-as 65002
spine-01(config-router)# neighbor 10.1.1.4 remote-as 65003
```

Define the three leaf neighbors (IPv6):
```
spine-01(config-router)# neighbor 2001:db8:1:1::0 remote-as 65001
spine-01(config-router)# neighbor 2001:db8:1:1::2 remote-as 65002
spine-01(config-router)# neighbor 2001:db8:1:1::4 remote-as 65003
```

Configure the IPv4 unicast address family:
```
spine-01(config-router)# address-family ipv4 unicast
spine-01(config-router-af)# network 4.4.4.4/32
spine-01(config-router-af)# neighbor 10.1.1.0 activate
spine-01(config-router-af)# neighbor 10.1.1.2 activate
spine-01(config-router-af)# neighbor 10.1.1.4 activate
spine-01(config-router-af)# maximum-paths 64
spine-01(config-router-af)# exit-address-family
```

Configure the IPv6 unicast address family:
```
spine-01(config-router)# address-family ipv6 unicast
spine-01(config-router-af)# network fc00:0:4::1/128
spine-01(config-router-af)# neighbor 2001:db8:1:1::0 activate
spine-01(config-router-af)# neighbor 2001:db8:1:1::0 route-map BGP-IPV6 in
spine-01(config-router-af)# neighbor 2001:db8:1:1::2 activate
spine-01(config-router-af)# neighbor 2001:db8:1:1::2 route-map BGP-IPV6 in
spine-01(config-router-af)# neighbor 2001:db8:1:1::4 activate
spine-01(config-router-af)# neighbor 2001:db8:1:1::4 route-map BGP-IPV6 in
spine-01(config-router-af)# maximum-paths 64
spine-01(config-router-af)# exit-address-family
spine-01(config-router)# exit
```

Configure the route maps:
```
spine-01(config)# route-map RM_SET_SRC permit 10
spine-01(config-route-map)# set src 4.4.4.4
spine-01(config-route-map)# exit

spine-01(config)# route-map RM_SET_SRC6 permit 10
spine-01(config-route-map)# set src fc00:0:4::1
spine-01(config-route-map)# exit

spine-01(config)# route-map BGP-IPV6 permit 20
spine-01(config-route-map)# set ipv6 next-hop prefer-global
spine-01(config-route-map)# exit
```

Bind protocol route maps (installs correct source address in kernel for BGP routes):
```
spine-01(config)# ip protocol bgp route-map RM_SET_SRC
spine-01(config)# ipv6 protocol bgp route-map RM_SET_SRC6
spine-01(config)# ip nht resolve-via-default
spine-01(config)# ipv6 nht resolve-via-default
spine-01(config)# end
```

Save the configuration to disk:
```
spine-01# copy running-config startup-config
```

This writes the running FRR configuration to `/etc/sonic/frr/bgpd.conf`.

Verify the saved file:
```
spine-01# exit
cisco@spine-01:~$ sudo cat /etc/sonic/frr/bgpd.conf
```

---

### Task 2: Configure BGP on leaf-01

SSH to leaf-01:
```
ssh cisco@172.20.2.100
```

Enter vtysh:
```
cisco@leaf-01:~$ sudo vtysh
```

Configure BGP — leaf-01 has a single neighbor (spine-01) in each address family:
```
leaf-01# configure terminal

leaf-01(config)# router bgp 65001
leaf-01(config-router)# bgp router-id 1.1.1.1
leaf-01(config-router)# bgp log-neighbor-changes
leaf-01(config-router)# no bgp ebgp-requires-policy
leaf-01(config-router)# no bgp default ipv4-unicast
leaf-01(config-router)# bgp bestpath as-path multipath-relax
leaf-01(config-router)# no bgp network import-check

leaf-01(config-router)# neighbor 10.1.1.1 remote-as 65000
leaf-01(config-router)# neighbor 2001:db8:1:1::1 remote-as 65000

leaf-01(config-router)# address-family ipv4 unicast
leaf-01(config-router-af)# network 1.1.1.1/32
leaf-01(config-router-af)# neighbor 10.1.1.1 activate
leaf-01(config-router-af)# maximum-paths 64
leaf-01(config-router-af)# exit-address-family

leaf-01(config-router)# address-family ipv6 unicast
leaf-01(config-router-af)# network fc00:0:1::1/128
leaf-01(config-router-af)# neighbor 2001:db8:1:1::1 activate
leaf-01(config-router-af)# neighbor 2001:db8:1:1::1 route-map BGP-IPV6 in
leaf-01(config-router-af)# maximum-paths 64
leaf-01(config-router-af)# exit-address-family
leaf-01(config-router)# exit

leaf-01(config)# route-map RM_SET_SRC permit 10
leaf-01(config-route-map)# set src 1.1.1.1
leaf-01(config-route-map)# exit

leaf-01(config)# route-map RM_SET_SRC6 permit 10
leaf-01(config-route-map)# set src fc00:0:1::1
leaf-01(config-route-map)# exit

leaf-01(config)# route-map BGP-IPV6 permit 20
leaf-01(config-route-map)# set ipv6 next-hop prefer-global
leaf-01(config-route-map)# exit

leaf-01(config)# ip protocol bgp route-map RM_SET_SRC
leaf-01(config)# ipv6 protocol bgp route-map RM_SET_SRC6
leaf-01(config)# ip nht resolve-via-default
leaf-01(config)# ipv6 nht resolve-via-default
leaf-01(config)# end

leaf-01# copy running-config startup-config
```

---

### Task 3: Configure BGP on leaf-02

SSH to leaf-02:
```
ssh cisco@172.20.2.101
```

Enter vtysh and configure BGP. leaf-02 is AS 65002, loopback 2.2.2.2, neighbor is the spine-01 Ethernet8 address:

```
cisco@leaf-02:~$ sudo vtysh

leaf-02# configure terminal

leaf-02(config)# router bgp 65002
leaf-02(config-router)# bgp router-id 2.2.2.2
leaf-02(config-router)# bgp log-neighbor-changes
leaf-02(config-router)# no bgp ebgp-requires-policy
leaf-02(config-router)# no bgp default ipv4-unicast
leaf-02(config-router)# bgp bestpath as-path multipath-relax
leaf-02(config-router)# no bgp network import-check

leaf-02(config-router)# neighbor 10.1.1.3 remote-as 65000
leaf-02(config-router)# neighbor 2001:db8:1:1::3 remote-as 65000

leaf-02(config-router)# address-family ipv4 unicast
leaf-02(config-router-af)# network 2.2.2.2/32
leaf-02(config-router-af)# neighbor 10.1.1.3 activate
leaf-02(config-router-af)# maximum-paths 64
leaf-02(config-router-af)# exit-address-family

leaf-02(config-router)# address-family ipv6 unicast
leaf-02(config-router-af)# network fc00:0:2::1/128
leaf-02(config-router-af)# neighbor 2001:db8:1:1::3 activate
leaf-02(config-router-af)# neighbor 2001:db8:1:1::3 route-map BGP-IPV6 in
leaf-02(config-router-af)# maximum-paths 64
leaf-02(config-router-af)# exit-address-family
leaf-02(config-router)# exit

leaf-02(config)# route-map RM_SET_SRC permit 10
leaf-02(config-route-map)# set src 2.2.2.2
leaf-02(config-route-map)# exit

leaf-02(config)# route-map RM_SET_SRC6 permit 10
leaf-02(config-route-map)# set src fc00:0:2::1
leaf-02(config-route-map)# exit

leaf-02(config)# route-map BGP-IPV6 permit 20
leaf-02(config-route-map)# set ipv6 next-hop prefer-global
leaf-02(config-route-map)# exit

leaf-02(config)# ip protocol bgp route-map RM_SET_SRC
leaf-02(config)# ipv6 protocol bgp route-map RM_SET_SRC6
leaf-02(config)# ip nht resolve-via-default
leaf-02(config)# ipv6 nht resolve-via-default
leaf-02(config)# end

leaf-02# copy running-config startup-config
```

---

### Task 4: Exercise — Configure BGP on leaf-03

Using the pattern established in Tasks 2 and 3, configure BGP on **leaf-03** independently.

**leaf-03 parameters:**

| Parameter          | Value                 |
|--------------------|-----------------------|
| ASN                | 65003                 |
| BGP Router-ID      | 3.3.3.3               |
| IPv4 Loopback      | 3.3.3.3/32            |
| IPv6 Loopback      | fc00:0:3::1/128       |
| IPv4 Neighbor (spine-01 Ethernet16) | 10.1.1.5  |
| IPv6 Neighbor (spine-01 Ethernet16) | 2001:db8:1:1::5 |
| Spine ASN          | 65000                 |

SSH to leaf-03:
```
ssh cisco@172.20.2.102
```

Configure BGP following the same steps as leaf-01 and leaf-02. When complete, save your configuration with `copy running-config startup-config`.

> **Verification check:** After configuring leaf-03, confirm the BGP session comes up by running `show bgp summary` on both leaf-03 and spine-01. You should see `Established` state and prefix counts greater than zero.

<details>
<summary>Solution (click to expand)</summary>

```
cisco@leaf-03:~$ sudo vtysh

leaf-03# configure terminal

leaf-03(config)# router bgp 65003
leaf-03(config-router)# bgp router-id 3.3.3.3
leaf-03(config-router)# bgp log-neighbor-changes
leaf-03(config-router)# no bgp ebgp-requires-policy
leaf-03(config-router)# no bgp default ipv4-unicast
leaf-03(config-router)# bgp bestpath as-path multipath-relax
leaf-03(config-router)# no bgp network import-check

leaf-03(config-router)# neighbor 10.1.1.5 remote-as 65000
leaf-03(config-router)# neighbor 2001:db8:1:1::5 remote-as 65000

leaf-03(config-router)# address-family ipv4 unicast
leaf-03(config-router-af)# network 3.3.3.3/32
leaf-03(config-router-af)# neighbor 10.1.1.5 activate
leaf-03(config-router-af)# maximum-paths 64
leaf-03(config-router-af)# exit-address-family

leaf-03(config-router)# address-family ipv6 unicast
leaf-03(config-router-af)# network fc00:0:3::1/128
leaf-03(config-router-af)# neighbor 2001:db8:1:1::5 activate
leaf-03(config-router-af)# neighbor 2001:db8:1:1::5 route-map BGP-IPV6 in
leaf-03(config-router-af)# maximum-paths 64
leaf-03(config-router-af)# exit-address-family
leaf-03(config-router)# exit

leaf-03(config)# route-map RM_SET_SRC permit 10
leaf-03(config-route-map)# set src 3.3.3.3
leaf-03(config-route-map)# exit

leaf-03(config)# route-map RM_SET_SRC6 permit 10
leaf-03(config-route-map)# set src fc00:0:3::1
leaf-03(config-route-map)# exit

leaf-03(config)# route-map BGP-IPV6 permit 20
leaf-03(config-route-map)# set ipv6 next-hop prefer-global
leaf-03(config-route-map)# exit

leaf-03(config)# ip protocol bgp route-map RM_SET_SRC
leaf-03(config)# ipv6 protocol bgp route-map RM_SET_SRC6
leaf-03(config)# ip nht resolve-via-default
leaf-03(config)# ipv6 nht resolve-via-default
leaf-03(config)# end

leaf-03# copy running-config startup-config
```

</details>

---

### Verify Saved FRR Configuration

On any node, confirm the FRR configuration was saved correctly to disk:

```
cisco@leaf-01:~$ show startupconfiguration bgp
```

Or directly view the file:
```
cisco@leaf-01:~$ sudo cat /etc/sonic/frr/bgpd.conf
```

You can also view the running FRR configuration from SONiC CLI:
```
cisco@leaf-01:~$ show run bgp
```


## BGP Verification

### BGP Summary

The `show bgp summary` command gives a quick overview of all BGP neighbors and their state. Run this on **leaf-01**:

```
cisco@leaf-01:~$ show bgp summary
```

Expected output:
```
IPv4 Unicast Summary (VRF default):
BGP router identifier 1.1.1.1, local AS number 65001 vrf-id 0
BGP table version 4
RIB entries 7, using 1288 bytes of memory
Peers 1, using 724 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.1.1.1        4      65000       135       132        0    0    0 00:02:18            3        1 N/A

Total number of neighbors 1

IPv6 Unicast Summary (VRF default):
BGP router identifier 1.1.1.1, local AS number 65001 vrf-id 0
BGP table version 4
RIB entries 7, using 1288 bytes of memory
Peers 1, using 724 KiB of memory

Neighbor              V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
2001:db8:1:1::1       4      65000       135       132        0    0    0 00:02:18            3        1 N/A

Total number of neighbors 1
```

Key fields to observe:
- **State/PfxRcd**: Should show a number (not `Idle`, `Active`, or `Connect`) — this is how many prefixes were received from the neighbor
- **PfxSnt**: Prefixes sent to the neighbor
- **Up/Down**: Session uptime — confirms the session is established

Now run the same command on **spine-01** to see all three leaf neighbors:

```
cisco@spine-01:~$ show bgp summary
```

Expected output shows three neighbors each receiving 1 prefix (their respective loopback) and sending 4 prefixes (spine loopback + 3 leaf loopbacks):
```
IPv4 Unicast Summary (VRF default):
BGP router identifier 4.4.4.4, local AS number 65000 vrf-id 0
BGP table version 7
RIB entries 7, using 1288 bytes of memory
Peers 3, using 2171 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.1.1.0        4      65001       218       215        0    0    0 00:05:31            1        4 N/A
10.1.1.2        4      65002       201       198        0    0    0 00:04:47            1        4 N/A
10.1.1.4        4      65003       187       185        0    0    0 00:04:12            1        4 N/A

Total number of neighbors 3
```

### BGP Neighbor Detail

For detailed session information including capabilities, timers, and message counters:

```
cisco@leaf-01:~$ show bgp neighbors 10.1.1.1
```

Key sections to review in the output:

```
BGP neighbor is 10.1.1.1, remote AS 65000, local AS 65001, external link
  BGP version 4, remote router ID 4.4.4.4
  BGP state = Established, up for 00:05:31
  Last read 00:00:30, Last write 00:00:30
  Hold time is 9, keepalive interval is 3 seconds
  ...
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Extended Message: advertised and received
    AddPath:
      IPv4 Unicast: RX advertised, TX advertised and received
    Route refresh: advertised and received(old & new)
    Address Family IPv4 Unicast: advertised and received
    Address Family IPv6 Unicast: advertised and received
  ...
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  1          1
    Notifications:          0          0
    Updates:                4          3
    Keepalives:           218        215
    Route Refresh:          0          0
    Capability:             0          0
    Total:                223        219
  ...
  Local host: 10.1.1.0, Local port: 51234
  Foreign host: 10.1.1.1, Foreign port: 179
```

Observe:
- **BGP state = Established** — session is up
- **4 Byte AS** capability is negotiated
- **Address Family IPv4 Unicast** and **IPv6 Unicast** are both advertised and received
- TCP session uses port 179 (BGP standard)

### BGP IPv4 Routing Table

View the BGP RIB (Routing Information Base) for IPv4:

```
cisco@leaf-01:~$ show bgp ipv4 unicast
```

Expected output:
```
BGP table version is 4, local router ID is 1.1.1.1, vrf id 0
Default local pref 100, local AS 65001
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, N invalid, U unknown

   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.1/32       0.0.0.0                  0         32768 i
*> 2.2.2.2/32       10.1.1.1                 0             0 65000 65002 i
*> 3.3.3.3/32       10.1.1.1                 0             0 65000 65003 i
*> 4.4.4.4/32       10.1.1.1                 0             0 65000 i

Displayed  4 routes and 4 total paths
```

Observe:
- `1.1.1.1/32` — locally originated (weight 32768, next-hop 0.0.0.0)
- `2.2.2.2/32` — learned via spine-01 (10.1.1.1), AS-PATH shows `65000 65002`
- `3.3.3.3/32` — learned via spine-01, AS-PATH shows `65000 65003`
- `4.4.4.4/32` — the spine's own loopback, AS-PATH shows `65000`

The AS-PATH is critical: leaf-01 (AS 65001) sees that to reach leaf-02 (AS 65002), the path transits through spine-01 (AS 65000). This is the normal eBGP transit behavior.

View specific prefix details including all paths:
```
cisco@leaf-01:~$ show bgp ipv4 unicast 2.2.2.2/32
```

Expected output:
```
BGP routing table entry for 2.2.2.2/32, version 2
Paths: (1 available, best #1, table default)
  Advertised to non peer-group peers:
  10.1.1.1
  65000 65002
    10.1.1.1 from 10.1.1.1 (4.4.4.4)
      Origin IGP, metric 0, localpref 100, weight 0, valid, external, best (First path received)
      Last update: Mon Apr 13 12:05:22 2026
```

### BGP IPv6 Routing Table

```
cisco@leaf-01:~$ show bgp ipv6 unicast
```

Expected output:
```
BGP table version is 4, local router ID is 1.1.1.1, vrf id 0
Default local pref 100, local AS 65001
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed

   Network                   Next Hop              Metric LocPrf Weight Path
*> fc00:0:1::1/128           ::                       0         32768 i
*> fc00:0:2::1/128           2001:db8:1:1::1          0             0 65000 65002 i
*> fc00:0:3::1/128           2001:db8:1:1::1          0             0 65000 65003 i
*> fc00:0:4::1/128           2001:db8:1:1::1          0             0 65000 i

Displayed  4 routes and 4 total paths
```

### IPv4 Routing Table

Confirm BGP routes are installed in the kernel routing table:

```
cisco@leaf-01:~$ show ip route
```

Expected output:
```
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup

B>* 2.2.2.2/32 [20/0] via 10.1.1.1, Ethernet0, weight 1, 00:05:31
B>* 3.3.3.3/32 [20/0] via 10.1.1.1, Ethernet0, weight 1, 00:04:12
B>* 4.4.4.4/32 [20/0] via 10.1.1.1, Ethernet0, weight 1, 00:05:31
C>* 10.1.1.0/31 is directly connected, Ethernet0, 02:14:05
C>* 1.1.1.1/32 is directly connected, Loopback0, 02:14:05
```

Key codes:
- `B` = BGP-learned route
- `C` = Connected route
- `>*` = Selected best route, installed in FIB

### IPv6 Routing Table

```
cisco@leaf-01:~$ show ipv6 route
```

### End-to-End Reachability Test

Verify loopback reachability across the fabric. From **leaf-01**, ping the loopback of every other device using the local loopback as the source:

```
cisco@leaf-01:~$ ping 2.2.2.2 -I 1.1.1.1 -c 3
cisco@leaf-01:~$ ping 3.3.3.3 -I 1.1.1.1 -c 3
cisco@leaf-01:~$ ping 4.4.4.4 -I 1.1.1.1 -c 3
```

Expected output (example for 2.2.2.2):
```
PING 2.2.2.2 (2.2.2.2) from 1.1.1.1 : 56(84) bytes of data.
64 bytes from 2.2.2.2: icmp_seq=1 ttl=63 time=2.14 ms
64 bytes from 2.2.2.2: icmp_seq=2 ttl=63 time=1.98 ms
64 bytes from 2.2.2.2: icmp_seq=3 ttl=63 time=2.01 ms

--- 2.2.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.980/2.043/2.140/0.069 ms
```

Notice the `ttl=63` (decremented by 1 at the spine) confirming the packet transited through spine-01.

Test IPv6 loopback reachability:
```
cisco@leaf-01:~$ ping6 fc00:0:2::1 -I fc00:0:1::1 -c 3
cisco@leaf-01:~$ ping6 fc00:0:3::1 -I fc00:0:1::1 -c 3
cisco@leaf-01:~$ ping6 fc00:0:4::1 -I fc00:0:1::1 -c 3
```

### BGP Route Advertisement Verification

Confirm what leaf-01 is advertising to spine-01:

```
cisco@leaf-01:~$ show bgp neighbors 10.1.1.1 advertised-routes
```

Expected output:
```
BGP table version is 4, local router ID is 1.1.1.1, vrf id 0
Default local pref 100, local AS 65001

   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.1/32       0.0.0.0                  0         32768 i

Total number of prefixes 1
```

Confirm what routes leaf-01 has received from spine-01:

```
cisco@leaf-01:~$ show bgp neighbors 10.1.1.1 received-routes
```

### Understanding the BGP Output in Detail

Let's decode the full BGP summary output field by field on **spine-01**:

```
cisco@spine-01:~$ show bgp summary
```

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 4.4.4.4, local AS number 65000 vrf-id 0
BGP table version 7
RIB entries 7, using 1288 bytes of memory
Peers 3, using 2171 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.1.1.0        4      65001       218       215        0    0    0 00:05:31            1        4 N/A
10.1.1.2        4      65002       201       198        0    0    0 00:04:47            1        4 N/A
10.1.1.4        4      65003       187       185        0    0    0 00:04:12            1        4 N/A
```

| Field          | Meaning                                                                 |
|----------------|-------------------------------------------------------------------------|
| `V`            | BGP version — always 4 (BGP-4)                                         |
| `AS`           | Remote AS number                                                        |
| `MsgRcvd`      | Total BGP messages received (OPEN + UPDATE + KEEPALIVE combined)       |
| `MsgSent`      | Total BGP messages sent                                                 |
| `TblVer`       | BGP table version at which this neighbor was last updated               |
| `InQ / OutQ`   | Number of messages queued to be processed inbound / outbound           |
| `Up/Down`      | How long the session has been in its current state                     |
| `State/PfxRcd` | If Established: number of prefixes received. If not: current FSM state |
| `PfxSnt`       | Number of prefixes advertised to this peer                              |

**Reading `State/PfxRcd`:**
- A number (e.g., `1`) → session is Established, received that many prefixes
- `Idle` → session is administratively or automatically down
- `Active` → BGP is retrying TCP connection
- `Connect` → TCP SYN sent, waiting for response
- `OpenSent` → TCP up, OPEN sent, waiting for peer's OPEN
- `OpenConfirm` → OPENs exchanged, waiting for KEEPALIVE

### BGP AS-PATH Analysis

Use AS-PATH regular expressions to filter and analyze the BGP table on **spine-01**:

```
cisco@spine-01:~$ sudo vtysh -c "show bgp ipv4 unicast regexp ^65001"
```

This shows only routes whose AS-PATH starts with AS 65001 (i.e., routes originated by leaf-01).

Other useful AS-PATH regex patterns:

| Regex Pattern  | Matches                                          |
|----------------|--------------------------------------------------|
| `^$`           | Locally originated routes (empty AS_PATH)        |
| `^65001$`      | Routes originated in AS 65001, one hop away      |
| `^65000`       | Routes that transited through AS 65000 (spine)   |
| `_65002_`      | Routes that passed through AS 65002 anywhere     |
| `.*`           | All routes (wildcard)                            |

### Observe BGP Session Establishment in Real Time

To watch a BGP session come up from scratch, clear a session and observe the state transitions:

```
cisco@leaf-01:~$ sudo vtysh
leaf-01# clear bgp 10.1.1.1
```

This sends a NOTIFICATION to the peer and resets the TCP session. In another terminal, watch the session state:

```
cisco@leaf-01:~$ watch -n 1 "sudo vtysh -c 'show bgp summary' 2>/dev/null | grep 10.1.1.1"
```

You should see the session cycle through: `Active` → `Connect` → `OpenSent` → `OpenConfirm` → `1` (Established with 1 prefix received).

### BGP Packet Capture

To observe actual BGP messages on the wire, use tcpdump. In one terminal:

```
cisco@leaf-01:~$ sudo tcpdump -i Ethernet0 tcp port 179 -v
```

In another terminal, reset the BGP session:
```
cisco@leaf-01:~$ sudo vtysh -c "clear bgp 10.1.1.1"
```

You will see the TCP 3-way handshake followed by BGP OPEN messages, then KEEPALIVE exchange, then UPDATE messages carrying the routes.

To decode BGP messages in full detail:
```
cisco@leaf-01:~$ sudo tcpdump -i Ethernet0 tcp port 179 -w /tmp/bgp_capture.pcap
```

### BGP Route Filtering Debug

To see exactly which routes are being accepted or rejected by route-maps, enable BGP debug:

```
cisco@leaf-01:~$ sudo vtysh
leaf-01# debug bgp updates in
leaf-01# debug bgp updates out
```

> **Warning:** Debug logging can be very verbose on a busy router. Always disable after use with `no debug bgp updates`.

View the debug output in the FRR log:
```
cisco@leaf-01:~$ docker exec bgp tail -f /var/log/frr/bgpd.log
```

### Check Next-Hop Tracking State

Verify that FRR's NHT (Next-Hop Tracking) considers the spine's IP as reachable:

```
cisco@leaf-01:~$ sudo vtysh -c "show bgp nexthop"
```

Expected output:
```
Current BGP nexthop cache:
 10.1.1.1 valid [IGP metric 0], #paths 3
  gate via 10.1.1.1, Ethernet0
  Last update: Mon Apr 13 12:05:20 2026
```

The `valid` status confirms that zebra has verified the next-hop 10.1.1.1 is reachable via a connected route on Ethernet0.

### Additional Verification Commands

| Command | Description |
|---------|-------------|
| `show bgp ipv4 unicast community` | Show routes with BGP communities |
| `show bgp ipv4 unicast statistics` | BGP RIB statistics |
| `show bgp memory` | BGP memory usage per peer and table |
| `show bgp vrfs` | BGP VRF table summary |
| `show ip route summary` | Routing table summary by protocol |
| `show bgp ipv4 unicast regexp ^65000` | Filter BGP table by AS-PATH regex |
| `show bgp neighbors X advertised-routes` | Prefixes sent to neighbor X |
| `show bgp neighbors X received-routes` | All prefixes received from X (before policy) |
| `show bgp neighbors X routes` | Prefixes received and accepted from X |
| `show bgp nexthop` | NHT state — validity of all tracked next-hops |
| `show bgp update-groups` | BGP update group membership |
| `show bgp peer-group` | Peer group configuration |

### BGP Troubleshooting Reference

| Symptom | Likely Cause | Where to Check |
|---------|-------------|----------------|
| Peer stuck in `Active` | TCP unreachable | `ping 10.1.1.1`; check `show interface status` |
| Peer stuck in `Idle` | Policy required / admin down | Check `no bgp ebgp-requires-policy`; `show bgp neighbors X` for shutdown message |
| Session up but no prefixes | `activate` missing in address-family | `show bgp neighbors X` — check "Address family IPv4 Unicast: not advertised" |
| Route in BGP RIB but not in `ip route` | Next-hop unreachable | `show bgp nexthop`; verify NHT shows `valid` |
| Route in `ip route` but not in hardware | SAI/orchagent issue | Check `redis-cli -n 1 keys "*ROUTE*"` in ASIC_DB |
| `State/PfxRcd` shows 0 | Peer has no matching prefix | Verify `network` statement on the remote peer |
| NOTIFICATION received | Misconfiguration | Check `docker exec bgp cat /var/log/frr/bgpd.log` for error code |


## Redis Database Verification

### SONiC Redis Architecture

SONiC uses Redis as the central message bus and state store. Multiple Redis databases (selected by index number) serve different subsystems:

| DB Index | Name          | Contents                                        |
|----------|---------------|-------------------------------------------------|
| 0        | APPL_DB       | Applied configuration: routes, neighbors, interfaces |
| 1        | ASIC_DB       | ASIC programming state (SAI objects)            |
| 2        | COUNTERS_DB   | Interface and ACL counters                      |
| 3        | LOGLEVEL_DB   | Daemon log level settings                       |
| 4        | CONFIG_DB     | Startup configuration (mirrors config_db.json)  |
| 5        | PFC_WD_DB     | PFC watchdog                                    |
| 6        | STATE_DB      | Runtime operational state                       |

When FRR's `bgpd` installs a route, the flow is:

```
bgpd → zebra → kernel FIB → fpmsyncd → APPL_DB:ROUTE_TABLE → orchagent → ASIC_DB → ASIC
```

The Redis database lets us inspect each step of this pipeline.

### Querying Routes in APPL_DB

List all BGP-installed routes in the application database:

```
cisco@leaf-01:~$ redis-cli -n 0 keys "ROUTE_TABLE:*"
```

Expected output:
```
1) "ROUTE_TABLE:2.2.2.2/32"
2) "ROUTE_TABLE:3.3.3.3/32"
3) "ROUTE_TABLE:4.4.4.4/32"
```

> **Note:** The locally connected routes and loopback are typically not in ROUTE_TABLE — they are in INTF_TABLE and NEIGH_TABLE respectively.

Get the full route entry for a specific prefix:

```
cisco@leaf-01:~$ redis-cli -n 0 hgetall "ROUTE_TABLE:2.2.2.2/32"
```

Expected output:
```
1) "nexthop"
2) "10.1.1.1"
3) "ifname"
4) "Ethernet0"
5) "blackhole"
6) "false"
```

This shows that the route to 2.2.2.2/32 is programmed with next-hop 10.1.1.1 via Ethernet0 — exactly matching what FRR resolved.

### Querying All Route Entries

Get all BGP routes and their attributes in a single pass:

```
cisco@leaf-01:~$ for key in $(redis-cli -n 0 keys "ROUTE_TABLE:*"); do
    echo "--- $key ---"
    redis-cli -n 0 hgetall "$key"
done
```

### Querying CONFIG_DB for Device Metadata

Confirm the BGP ASN as stored in CONFIG_DB:

```
cisco@leaf-01:~$ redis-cli -n 4 hgetall "DEVICE_METADATA|localhost"
```

Expected output:
```
 1) "hostname"
 2) "leaf-01"
 3) "bgp_asn"
 4) "65001"
 5) "docker_routing_config_mode"
 6) "split"
 7) "buffer_model"
 8) "traditional"
 9) "switch_type"
10) "switch"
11) "mac"
12) "02:42:ac:14:06:64"
```

This confirms that CONFIG_DB holds the `bgp_asn` and `docker_routing_config_mode` values loaded from `config_db.json`.

### Querying Interface Configuration in CONFIG_DB

View how Ethernet0's IP address is stored in CONFIG_DB:

```
cisco@leaf-01:~$ redis-cli -n 4 hgetall "INTERFACE|Ethernet0|10.1.1.0/31"
```

Expected output:
```
(empty array)
```

> **Note:** In SONiC, an empty hash for an interface IP key is valid — the key existence itself signals that the IP is configured. The IP is encoded in the key name itself (`INTERFACE|Ethernet0|10.1.1.0/31`).

List all interface keys:
```
cisco@leaf-01:~$ redis-cli -n 4 keys "INTERFACE|*"
```

Expected output:
```
1) "INTERFACE|Ethernet0"
2) "INTERFACE|Ethernet0|10.1.1.0/31"
3) "INTERFACE|Ethernet0|2001:db8:1:1::0/127"
4) "INTERFACE|Ethernet8"
```

### Querying BGP Neighbor State in STATE_DB

Check the operational state of BGP neighbors as seen by SONiC's state database:

```
cisco@leaf-01:~$ redis-cli -n 6 keys "BGP_NEIGHBOR_TABLE|*"
```

Expected output:
```
1) "BGP_NEIGHBOR_TABLE|default|10.1.1.1"
2) "BGP_NEIGHBOR_TABLE|default|2001:db8:1:1::1"
```

Get the state of the IPv4 BGP neighbor:
```
cisco@leaf-01:~$ redis-cli -n 6 hgetall "BGP_NEIGHBOR_TABLE|default|10.1.1.1"
```

Expected output:
```
1) "state"
2) "Established"
3) "local_port"
4) "51234"
5) "remote_port"
6) "179"
7) "local_addr"
8) "10.1.1.0"
9) "peer_addr"
10) "10.1.1.1"
```

### Querying the Loopback Interface in APPL_DB

```
cisco@leaf-01:~$ redis-cli -n 0 keys "INTF_TABLE:Loopback0*"
```

Expected output:
```
1) "INTF_TABLE:Loopback0"
2) "INTF_TABLE:Loopback0:1.1.1.1/32"
3) "INTF_TABLE:Loopback0:fc00:0:1::1/128"
```

### Monitoring Route Installation

To watch routes being programmed into APPL_DB in real time (useful during troubleshooting), use the Redis `monitor` command:

```
cisco@leaf-01:~$ redis-cli -n 0 monitor
```

> Press `Ctrl+C` to stop monitoring. This is equivalent to a debug session — use it when you need to watch live state changes.

### Verify Route in ASIC_DB (Hardware Programming)

After the route flows through APPL_DB and orchagent, it should appear in ASIC_DB as a SAI route entry. This confirms the route has been sent to the hardware driver:

```
cisco@leaf-01:~$ redis-cli -n 1 keys "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:*"
```

Expected output (showing the BGP-learned routes alongside connected routes):
```
1) "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{\"dest\":\"2.2.2.2/32\",\"switch_id\":\"oid:0x21000000000000\",\"vr_id\":\"oid:0x3000000000022\"}"
2) "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{\"dest\":\"3.3.3.3/32\",\"switch_id\":\"oid:0x21000000000000\",\"vr_id\":\"oid:0x3000000000022\"}"
3) "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{\"dest\":\"4.4.4.4/32\",\"switch_id\":\"oid:0x21000000000000\",\"vr_id\":\"oid:0x3000000000022\"}"
```

Get the details of one ASIC route entry to see the next-hop object ID:
```
cisco@leaf-01:~$ redis-cli -n 1 hgetall "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{\"dest\":\"2.2.2.2/32\",\"switch_id\":\"oid:0x21000000000000\",\"vr_id\":\"oid:0x3000000000022\"}"
```

Expected output:
```
1) "SAI_ROUTE_ENTRY_ATTR_NEXT_HOP_ID"
2) "oid:0x400000000061e"
```

The `oid:` (Object ID) is the SAI handle for the next-hop group object. You can further inspect this next-hop:
```
cisco@leaf-01:~$ redis-cli -n 1 hgetall "ASIC_STATE:SAI_OBJECT_TYPE_NEXT_HOP:oid:0x400000000061e"
```

This level of inspection confirms the full path: BGP RIB → Linux kernel → APPL_DB → ASIC_DB → hardware FIB.

### Verify the Full Pipeline in One Pass

Here is a consolidated verification sequence that walks the entire SONiC route pipeline for `2.2.2.2/32`:

```bash
echo "=== 1. FRR BGP RIB ===" 
sudo vtysh -c "show bgp ipv4 unicast 2.2.2.2/32"

echo "=== 2. Linux Kernel FIB ===" 
ip route show 2.2.2.2/32

echo "=== 3. SONiC APPL_DB (fpmsyncd output) ===" 
redis-cli -n 0 hgetall "ROUTE_TABLE:2.2.2.2/32"

echo "=== 4. SONiC ASIC_DB (orchagent output) ===" 
redis-cli -n 1 keys "ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:*2.2.2.2*"
```

Each step should show the route — if any step is missing the route, you have found where the pipeline is broken.

### Watching CONFIG_DB Populated from config_db.json

To understand how the interface configuration from `config_db.json` is reflected in CONFIG_DB:

```
cisco@leaf-01:~$ redis-cli -n 4 keys "LOOPBACK_INTERFACE|*"
```

Expected output:
```
1) "LOOPBACK_INTERFACE|Loopback0"
2) "LOOPBACK_INTERFACE|Loopback0|1.1.1.1/32"
3) "LOOPBACK_INTERFACE|Loopback0|fc00:0:1::1/128"
```

The key pattern `TABLE|interface|prefix` is SONiC's standard encoding: the interface is identified by name and the IP prefix is encoded in the key itself. The hash value is often empty — the key's existence is the data.

Verify the interface-level metadata:
```
cisco@leaf-01:~$ redis-cli -n 4 hgetall "PORT|Ethernet0"
```

Expected output:
```
 1) "alias"
 2) "etp0"
 3) "index"
 4) "0"
 5) "lanes"
 6) "2304,2305,2306,2307,2308,2309,2310,2311"
 7) "mtu"
 8) "9100"
 9) "speed"
10) "400000"
11) "admin_status"
12) "up"
```

This directly mirrors the `PORT` table in `config_db.json` from Lab 1, confirming the config load pipeline worked correctly.

### Redis Subscribe — Watch Live Route Events

Redis supports a publish/subscribe model. SONiC uses this to notify processes of table changes. You can observe live events as routes are programmed:

```
cisco@leaf-01:~$ redis-cli -n 0 psubscribe "__keyevent@0__:hset"
```

In another terminal, reset the BGP session:
```
cisco@leaf-01:~$ sudo vtysh -c "clear bgp 10.1.1.1"
```

You will see Redis keyevent notifications fired each time fpmsyncd writes a route to APPL_DB — one event per prefix as the BGP table is re-established.

Press `Ctrl+C` to stop subscribing.

### Redis CLI Reference

| Command | Description |
|---------|-------------|
| `redis-cli -n 0 keys "*"` | List all keys in APPL_DB |
| `redis-cli -n 0 hgetall "KEY"` | Get all fields of a hash key |
| `redis-cli -n 1 keys "ASIC_STATE*ROUTE*"` | Route entries in ASIC_DB |
| `redis-cli -n 4 keys "BGP*"` | Find BGP-related keys in CONFIG_DB |
| `redis-cli -n 6 keys "*"` | List all keys in STATE_DB |
| `redis-cli -n 0 dbsize` | Count all keys in APPL_DB |
| `redis-cli info keyspace` | Show key counts across all databases |
| `redis-cli -n 0 monitor` | Live stream of all APPL_DB operations |
| `redis-cli -n 0 psubscribe "__keyevent@0__:hset"` | Subscribe to APPL_DB write events |


## End of Lab 2

You have successfully:
- Configured eBGP sessions between three leaves and one spine using FRR's `vtysh`
- Advertised loopback prefixes across the BGP fabric
- Verified BGP session state, learned prefixes, and AS-PATH attributes
- Confirmed end-to-end IPv4 and IPv6 reachability across the fabric
- Inspected how BGP routing state is stored and propagated through SONiC's Redis database pipeline

Lab 2 is completed, please proceed to [Lab 3](https://github.com/cisco-asp-web/SONiC/blob/main/lab_3/lab_3-guide.md)
