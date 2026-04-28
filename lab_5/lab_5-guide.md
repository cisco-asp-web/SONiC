# Lab 5 SONiC and FRR, Building a VXLAN EVPN Fabric [100 Min]

---

## Introduction

This lab walks you through building a VXLAN EVPN overlay on top of an existing eBGP underlay fabric. The underlay (interface IPs, BGP sessions, loopback reachability) was configured in Lab 2 (SONiC and FRR – Building a BGP Fabric) — this guide starts by verifying it is healthy, then builds the overlay from there. The spine acts as an EVPN route server, reflecting routes between leaves without modifying BGP attributes.

Two leaves (Leaf1 and Leaf2) share a multihomed host via an active-active LACP PortChannel and a common Ethernet Segment Identifier (ESI). A third leaf (Leaf3) has two single-homed hosts in separate subnets. By the end of the lab you will have a working VXLAN fabric with L2 extension, L3 routing (symmetric IRB), and EVPN multihoming.

**SONiC version:** FRR 8.5.x, `split-unified` routing mode

---

## Table of Contents

1. [Lab Objectives](#lab-objectives)
2. [Lab Access](#lab-access)
3. [Topology & Reference](#topology--reference)
4. [Task 1 — Prerequisites](#task-1--prerequisites)
5. [Task 2 — Verify Underlay](#task-2--verify-underlay)
6. [Task 3 — SONiC Overlay Configuration](#task-3--sonic-overlay-configuration)
7. [Task 4 — FRR Overlay Configuration](#task-4--frr-overlay-configuration)
8. [Task 5 — Verification](#task-5--verification)
   - [5.1 VXLAN Tunnel State](#51-vxlan-tunnel-state)
   - [5.2 EVPN Control Plane](#52-evpn-control-plane)
   - [5.3 Multihoming](#53-multihoming-verification)
   - [5.4 L3VNI / Symmetric IRB](#54-l3vni--symmetric-irb)
   - [5.5 Linux Dataplane](#55-linux-kernel--dataplane)
   - [5.6 Platform NPU](#56-platform-npu-verification)
9. [Appendix A — Underlay Configuration](#appendix-a--underlay-configuration)

---

## Lab Objectives

By completing this lab you will be able to:

- [ ] Verify the eBGP underlay is healthy and loopbacks are reachable (required for VXLAN tunnels)
- [ ] Create VXLAN tunnel objects (VTEPs) and map VLANs to VNIs
- [ ] Configure a VRF with an L3VNI for symmetric IRB
- [ ] Configure a Static Anycast Gateway (SAG) with the same IP on all leaves
- [ ] Build a PortChannel on Leaf1 and Leaf2 with a shared ESI for multihoming
- [ ] Configure the EVPN overlay BGP session (loopback-to-loopback) to the route server
- [ ] Verify EVPN route types 1 through 5 are exchanged correctly
- [ ] Confirm ECMP nexthop groups are created for the multihomed ESI
- [ ] Verify L3 routing across the fabric via Type-5 prefixes

---

## Lab Access

### Accessing SONiC Devices

From your container, SSH to any device using the pre-loaded aliases:

```bash
ssh leaf1       # password: password
ssh leaf2       # password: password
ssh leaf3       # password: password
ssh spine4      # password: password
```

### Accessing Hosts

Hosts are not directly accessible via SSH. Use the pre-defined shell functions from your container to send pings:

```bash
host12 ping -c3 <destination>     # Ping from Host12 (10.0.1.10/24, multihomed via Leaf1+Leaf2)
host3a ping -c3 <destination>     # Ping from Host3a (10.0.1.30/24, single-homed on Leaf3)
host3b ping -c3 <destination>     # Ping from Host3b (10.0.2.40/24, single-homed on Leaf3)
```

Try it now — ping the default gateway from each host:
```bash
host12 ping -c3 10.0.1.1
host3a ping -c3 10.0.1.1
host3b ping -c3 10.0.2.1
```

> **Note:** These pings will fail at this point — that's expected. The overlay is not configured yet. You will use these commands again after completing the configuration tasks.

### Host Addressing

| Host   | IP Address    | Gateway   | Connected To        |
|--------|---------------|-----------|---------------------|
| Host12 | 10.0.1.10/24  | 10.0.1.1  | Leaf1+Leaf2 (LACP)  |
| Host3a | 10.0.1.30/24  | 10.0.1.1  | Leaf3 Ethernet4     |
| Host3b | 10.0.2.40/24  | 10.0.2.1  | Leaf3 Ethernet8     |

---

## Topology & Reference

```
                         ┌─────────────────┐
                         │      Spine4     │
                         │   EVPN Route    │
                         │     Server      │
                         └──┬──────┬──────┬┘
                            │      │      │
                            │      │      │
              ┌─────────────┘      │      └─────────────┐
              │                    │                    │
       ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
       │    Leaf1    │      │    Leaf2    │      │    Leaf3    │
       │    AS 1     │      │    AS 2     │      │    AS 3     │
       └──────┬──────┘      └──────┬──────┘      └──┬───────┬──┘
              │                    │                 │       │
              └────────ESI-1───────┘                 │       │
                  Multihomed Host                Host3a   Host3b
                    (Hosts12)                   Eth4/V10  Eth8/V20
                                                  (Vrfa)   (Vrfa)
```

### Device Addressing

| Device | AS | Loopback  | Spine4-facing IP |
|--------|----|-----------|------------------|
| Spine4 | 4  | 4.4.4.4   | —                |
| Leaf1  | 1  | 1.1.1.1   | 1.4.1.4/24       |
| Leaf2  | 2  | 2.2.2.2   | 2.4.1.4/24       |
| Leaf3  | 3  | 3.3.3.3   | 3.4.1.4/24       |

### VNI / VLAN / VRF Assignments

| Purpose   | VLAN    | VNI  | VRF  | Leaves              |
|-----------|---------|------|------|---------------------|
| L2 subnet | Vlan10  | 1010 | Vrfa | Leaf1, Leaf2, Leaf3 |
| L2 subnet | Vlan20  | 2020 | Vrfa | Leaf3 only          |
| L3VNI     | Vlan900 | 9900 | Vrfa | Leaf1, Leaf2, Leaf3 |

### Tenant Networks & SAG

All subnets share the same VRF (Vrfa) and L3VNI (9900). The shared L3VNI handles inter-subnet routing across the fabric — Leaf1 and Leaf2 automatically receive Type-5 routes for Host3b's subnet without any additional config.

| Subnet | VLAN | SAG IP | SAG MAC | Leaves |
|--------|------|--------|---------|--------|
| 10.0.1.0/24 | Vlan10 | 10.0.1.1/24 | 0a:aa:aa:aa:aa:aa | All leaves |
| 10.0.2.0/24 | Vlan20 | 10.0.2.1/24 | — | Leaf3 only |

### Multihoming Reference (Leaf1 + Leaf2)

| Parameter     | Value                             |
|---------------|-----------------------------------|
| PortChannel   | PortChannel1                      |
| Access port   | Ethernet4                         |
| es-sys-mac    | 00:34:34:34:34:34                 |
| es-id         | 1                                 |
| ESI (derived) | 03:00:34:34:34:34:34:00:00:01     |
| DF winner     | Leaf1 (default pref 32767)        |
| non-DF        | Leaf2 (es-df-pref 100)            |

---

## Task 1 — Prerequisites

> **Applies to:** Leaf1, Leaf2, Leaf3 (not Spine4)
>
> All leaves must be in `split-unified` routing mode before any FRR configuration is applied. Without this, FRR and SONiC CLI manage separate routing stacks.

### Step 1.1 — Check current routing mode

Run on **Leaf1, Leaf2, and Leaf3** — check the output on each:

```bash
show runningconfiguration all | grep docker_routing_config_mode
```

If all three show `"split-unified"`, skip to Task 2. If absent or different, continue with Step 1.2.

### Step 1.2 — Apply `split-unified` mode

Run the following on **Leaf1, Leaf2, and Leaf3**. The commands are identical on each leaf.

```bash
cat > /tmp/routing_mode.json << 'EOF'
{
    "DEVICE_METADATA": {
        "localhost": {
            "docker_routing_config_mode": "split-unified"
        }
    }
}
EOF

sudo config load /tmp/routing_mode.json -y
sudo config save -y
sudo systemctl restart bgp
```

### Step 1.3 — Verify

Run on **Leaf1, Leaf2, and Leaf3**:

```bash
show runningconfiguration all | grep docker_routing_config_mode
# Expected: "docker_routing_config_mode": "split-unified"
```

---

## Task 2 — Verify Underlay

> **Goal:** Confirm the eBGP underlay from Lab 2 (SONiC and FRR – Building a BGP Fabric) is healthy. Each leaf must have an established BGP session with Spine4 and be able to ping all remote loopbacks. VXLAN tunnels require loopback-to-loopback IP reachability — nothing in this lab will work without it.
>
> If the underlay is **not** up, see [Appendix A — Underlay Configuration](#appendix-a--underlay-configuration) for the full configuration steps.

### Step 2.1 — Check BGP Neighbor State

Run on **each leaf** — verify the underlay peer is `Established` with a non-zero prefix count:

```bash
vtysh -c 'show bgp ipv4 unicast summary'
```

Expected output on **Leaf1** (Leaf2/Leaf3 will show their respective spine-facing IP):
```
BGP router identifier 1.1.1.1, local AS number 1 vrf-id 0
BGP table version 8
RIB entries 15, using 3360 bytes of memory
Peers 1, using 22 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
1.4.1.4         4          4        20        18        0    0    0 00:09:51            7        8 N/A

Total number of neighbors 1
```

If `State/PfxRcd` shows `Idle` or `Active` instead of a number, the BGP session is not established. Check [Appendix A](#appendix-a--underlay-configuration).

### Step 2.2 — Ping Remote Loopbacks

Run on **Leaf1** (repeat from Leaf2 and Leaf3 as well):

```bash
ping 2.2.2.2 -c3
ping 3.3.3.3 -c3
ping 4.4.4.4 -c3
```

Expected:
```
admin@Leaf1:~$ ping 3.3.3.3 -c3
PING 3.3.3.3 (3.3.3.3) 56(84) bytes of data.
64 bytes from 3.3.3.3: icmp_seq=1 ttl=63 time=150 ms
64 bytes from 3.3.3.3: icmp_seq=2 ttl=63 time=161 ms
64 bytes from 3.3.3.3: icmp_seq=3 ttl=63 time=81.2 ms

--- 3.3.3.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 81.178/130.437/160.536/35.116 ms
```

`ttl=63` confirms single-hop transit through the spine. All three remote loopbacks must return 0% packet loss.

> **Do not proceed to Task 3 until all loopback pings succeed.** If pings fail, see [Appendix A](#appendix-a--underlay-configuration).

---

## Task 3 — SONiC Overlay Configuration

> **Goal:** Create the VXLAN tunnel object (VTEP), map VLANs to VNIs, bind the L3VNI VLAN to the VRF, and configure the Static Anycast Gateway on Vlan10. Then connect host-facing ports.

### Step 3.1 — VXLAN Tunnel (VTEP)

Run on **Leaf1**:
```bash
sudo config vxlan add VTEP-10 1.1.1.1
sudo config vxlan evpn_nvo add evpn_nvo VTEP-10
```

Run on **Leaf2**:
```bash
sudo config vxlan add VTEP-10 2.2.2.2
sudo config vxlan evpn_nvo add evpn_nvo VTEP-10
```

Run on **Leaf3**:
```bash
sudo config vxlan add VTEP-10 3.3.3.3
sudo config vxlan evpn_nvo add evpn_nvo VTEP-10
```

### Step 3.2 — VLANs and VNI Mappings

Run on **Leaf1, Leaf2, and Leaf3** — commands are identical on each:
```bash
sudo config vlan add 10
sudo config vxlan map add VTEP-10 10 1010

sudo config vlan add 900
sudo config vxlan map add VTEP-10 900 9900
```

> Vlan900 is the L3VNI VLAN. Do **not** configure any IP address on its SVI — it is only used for VXLAN encapsulation of routed traffic.

### Step 3.3 — VRF and L3VNI Binding

Run on **Leaf1, Leaf2, and Leaf3** — commands are identical on each:
```bash
sudo config vrf add Vrfa
sudo config interface vrf bind Vlan900 Vrfa
sudo config vrf add_vrf_vni_map Vrfa 9900
```

> The `add_vrf_vni_map` command creates the VRF-to-VNI mapping in CONFIG_DB, which orchagent uses to program the L3VNI in the ASIC. Without it, `show vxlan vrfvnimap` will be empty even if FRR has `vni 9900` configured under the VRF. Both the SONiC-side mapping (this step) and the FRR-side mapping (Step 4.2) are required.

### Step 3.4 — Tenant SVI and Static Anycast Gateway

Run on **Leaf1, Leaf2, and Leaf3** — the same SAG IP (`10.0.1.1`) is intentionally configured on every leaf:
```bash
sudo config static-anycast-gateway mac_address add 0a:aa:aa:aa:aa:aa
sudo config interface vrf bind Vlan10 Vrfa
sudo config interface ip add Vlan10 10.0.1.1/24
sudo config vlan static-anycast-gateway enable 10
```

### Step 3.5 — Host-Facing Ports

#### Leaf1 and Leaf2 — Multihomed Host via PortChannel

Run on **Leaf1**:
```bash
sudo config portchannel add PortChannel1
sudo config portchannel member add PortChannel1 Ethernet4
sudo config vlan member add -u 10 PortChannel1
sudo config interface sys-mac add PortChannel1 00:34:34:34:34:34
sudo config save -y
```

Run on **Leaf2**:
```bash
sudo config portchannel add PortChannel1
sudo config portchannel member add PortChannel1 Ethernet4
sudo config vlan member add -u 10 PortChannel1
sudo config interface sys-mac add PortChannel1 00:34:34:34:34:34
sudo config save -y
```

#### Leaf3 — Single-Homed Hosts via Access Ports

Run on **Leaf3** — Host3a on Ethernet4 (Vlan10/Vrfa):
```bash
sudo config vlan member add -u 10 Ethernet4
```

### Step 3.6 — Host3b Subnet (Leaf3 only)

> Host3b connects to Leaf3 on Ethernet8 in a different subnet (Vlan20 / 10.0.2.0/24) but the **same VRF** (Vrfa).

Run on **Leaf3**:
```bash
sudo config vlan add 20
sudo config vxlan map add VTEP-10 20 2020
sudo config interface vrf bind Vlan20 Vrfa
sudo config interface ip add Vlan20 10.0.2.1/24
sudo config vlan static-anycast-gateway enable 20
sudo config vlan member add -u 20 Ethernet8
sudo config save -y
```

---

## Task 4 — FRR Overlay Configuration

> **Goal:** Configure the EVPN overlay BGP session from each leaf to the spine's loopback (as EVPN route server), enable EVPN MH global settings, bind the VRF VNI in FRR, configure the ESI on the PortChannels (Leaf1/Leaf2 only), and configure the spine as route server.

### Step 4.1 — Global EVPN MH Setting

Run on **Leaf1, Leaf2, and Leaf3** — enter `vtysh` first, then paste:
```
evpn mh redirect-off
```

### Step 4.2 — VRF VNI Binding in FRR

Run on **Leaf1, Leaf2, and Leaf3** — in `vtysh`:
```
vrf Vrfa
 vni 9900
exit-vrf
```

### Step 4.3 — ESI on PortChannel

> Only required on **Leaf1 and Leaf2**. Both must use the same `es-id` and `es-sys-mac`. Leaf3 requires no ESI configuration.

Run on **Leaf1** — no explicit `es-df-pref` (defaults to 32767, wins DF election):
```
interface PortChannel1
 evpn mh es-id 1
 evpn mh es-sys-mac 00:34:34:34:34:34
exit
```

Run on **Leaf2** — explicit `es-df-pref 100` (loses DF election to Leaf1):
```
interface PortChannel1
 evpn mh es-id 1
 evpn mh es-sys-mac 00:34:34:34:34:34
 evpn mh es-df-pref 100
exit
```

> The ESI is derived automatically as `03:<sys-mac>:<es-id>` →
> `03:00:34:34:34:34:34:00:00:01`. In DF election, **higher preference wins** —
> Leaf1's default 32767 beats Leaf2's configured 100.

### Step 4.4 — Overlay BGP Session (Leaf → Route Server)

Configure a loopback-to-loopback eBGP session to the spine's loopback (`4.4.4.4`) and enable EVPN address family.

Run on **Leaf1**:
```
router bgp 1
 neighbor 4.4.4.4 remote-as 4
 neighbor 4.4.4.4 ebgp-multihop 10
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family l2vpn evpn
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 route-map PASS in
  neighbor 4.4.4.4 route-map PASS out
  advertise-all-vni
  no use-es-l3nhg
  vni 1010
   route-target import 1010:1010
   route-target export 1010:1010
  exit-vni
 exit-address-family
exit
```

Run on **Leaf2**:
```
router bgp 2
 neighbor 4.4.4.4 remote-as 4
 neighbor 4.4.4.4 ebgp-multihop 10
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family l2vpn evpn
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 route-map PASS in
  neighbor 4.4.4.4 route-map PASS out
  advertise-all-vni
  no use-es-l3nhg
  vni 1010
   route-target import 1010:1010
   route-target export 1010:1010
  exit-vni
 exit-address-family
exit
```

Run on **Leaf3** — includes both VNI 1010 (Vlan10) and VNI 2020 (Vlan20) route-targets:
```
router bgp 3
 neighbor 4.4.4.4 remote-as 4
 neighbor 4.4.4.4 ebgp-multihop 10
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family l2vpn evpn
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 route-map PASS in
  neighbor 4.4.4.4 route-map PASS out
  advertise-all-vni
  no use-es-l3nhg
  vni 1010
   route-target import 1010:1010
   route-target export 1010:1010
  exit-vni
  vni 2020
   route-target import 2020:2020
   route-target export 2020:2020
  exit-vni
 exit-address-family
exit
```

### Step 4.5 — L3VNI / Symmetric IRB — VRF BGP

Configure the per-VRF BGP process to redistribute connected routes and enable EVPN Type-5 advertisement.

Run on **Leaf1**:
```
router bgp 1 vrf Vrfa
 bgp router-id 1.1.1.1
 bgp bestpath as-path multipath-relax
 !
 address-family ipv4 unicast
  redistribute connected
  rt vpn both 9900:9900
 exit-address-family
 !
 address-family l2vpn evpn
  no use-es-l3nhg
  advertise ipv4 unicast
  route-target import 9900:9900
  route-target export 9900:9900
 exit-address-family
exit
```

Run on **Leaf2**:
```
router bgp 2 vrf Vrfa
 bgp router-id 2.2.2.2
 bgp bestpath as-path multipath-relax
 !
 address-family ipv4 unicast
  redistribute connected
  rt vpn both 9900:9900
 exit-address-family
 !
 address-family l2vpn evpn
  no use-es-l3nhg
  advertise ipv4 unicast
  route-target import 9900:9900
  route-target export 9900:9900
 exit-address-family
exit
```

Run on **Leaf3**:
```
router bgp 3 vrf Vrfa
 bgp router-id 3.3.3.3
 bgp bestpath as-path multipath-relax
 !
 address-family ipv4 unicast
  redistribute connected
  rt vpn both 9900:9900
 exit-address-family
 !
 address-family l2vpn evpn
  no use-es-l3nhg
  advertise ipv4 unicast
  route-target import 9900:9900
  route-target export 9900:9900
 exit-address-family
exit
```

> Leaf3's Vrfa VRF BGP process is identical to Leaf1/Leaf2. The `redistribute connected` picks up both 10.0.1.0/24 (Vlan10) and 10.0.2.0/24 (Vlan20) and advertises them as Type-5 routes via L3VNI 9900. Leaf1 and Leaf2 import these automatically — no config changes needed on them.

### Step 4.6 — Route Server Configuration (Spine4 only)

The spine reflects EVPN routes between leaves using `route-server-client` so next-hops and AS paths are preserved exactly as originated.

Run on **Spine4**:
```
router bgp 4
 neighbor EVPN peer-group
 neighbor EVPN ebgp-multihop 10
 neighbor EVPN update-source Loopback0
 neighbor 1.1.1.1 remote-as 1
 neighbor 1.1.1.1 peer-group EVPN
 neighbor 2.2.2.2 remote-as 2
 neighbor 2.2.2.2 peer-group EVPN
 neighbor 3.3.3.3 remote-as 3
 neighbor 3.3.3.3 peer-group EVPN
 !
 address-family l2vpn evpn
  neighbor EVPN activate
  neighbor EVPN route-map PASS in
  neighbor EVPN route-map PASS out
  neighbor EVPN route-server-client
  neighbor EVPN attribute-unchanged as-path next-hop med
 exit-address-family
exit
```

> Without `route-server-client` and `attribute-unchanged`, the spine would
> rewrite the EVPN next-hop to its own loopback (`4.4.4.4`). Every leaf would
> then try to VXLAN-encapsulate traffic toward the spine, which is not a VTEP,
> causing all overlay traffic to be blackholed.

---

## Generate Traffic (Before Verification)

> Before running the verification commands, generate traffic from the hosts so that MAC addresses and ARP entries are learned across the fabric. Without traffic, the MAC and route tables will be empty.

Run these commands from your **container** (not from the SONiC devices):

**Step 1 — Hosts ping their default gateway** (populates local MAC/ARP):
```bash
host12 ping -c3 10.0.1.1
host3a ping -c3 10.0.1.1
host3b ping -c3 10.0.2.1
```

**Step 2 — Cross-VTEP L2 ping** (populates remote MAC tables via Type-2 routes):
```bash
host12 ping -c3 10.0.1.30
host3a ping -c3 10.0.1.10
```

**Step 3 — Cross-subnet L3 ping** (populates VRF routes and confirms symmetric IRB):
```bash
host12 ping -c3 10.0.2.40
host3b ping -c3 10.0.1.10
host3a ping -c3 10.0.2.40
```

All pings should return 0% packet loss. If any fail, go back and check the configuration steps before proceeding to verification.

---

## Task 5 — Verification

### 5.1 VXLAN Tunnel State

Verifies that the VTEP is correctly anchored to the loopback, that VLANs are mapped to the right VNIs, and that remote VTEPs have been discovered via EVPN Type-3 IMET routes. Also checks that remote MACs are being learned via Type-2 routes.

#### VTEP Source Interface

Run on **any leaf** — the source IP should match that leaf's loopback:
```
admin@Leaf1:~$ show vxlan interface
VTEP Information:

	VTEP Name : VTEP-10, SIP  : 1.1.1.1
	NVO Name  : evpn_nvo,  VTEP : VTEP-10
	Source interface  : Loopback0
```

#### Tunnel and VNI Mappings

Run on **Leaf1** (Leaf1/Leaf2 show two mappings; Leaf3 shows three including Vlan20):
```
admin@Leaf1:~$ show vxlan tunnel
vxlan tunnel name    source ip    destination ip    tunnel map name    tunnel map mapping(vni -> vlan)
-------------------  -----------  ----------------  -----------------  ---------------------------------
VTEP-10              1.1.1.1                        map_1010_Vlan10    1010 -> Vlan10
                                                    map_9900_Vlan900   9900 -> Vlan900
```

Run on **Leaf3** (includes Vlan20/VNI 2020 for Host3b):
```
admin@Leaf3:~$ show vxlan tunnel
vxlan tunnel name    source ip    destination ip    tunnel map name    tunnel map mapping(vni -> vlan)
-------------------  -----------  ----------------  -----------------  ---------------------------------
VTEP-10              3.3.3.3                        map_1010_Vlan10    1010 -> Vlan10
                                                    map_2020_Vlan20    2020 -> Vlan20
                                                    map_9900_Vlan900   9900 -> Vlan900
```

#### VLAN→VNI Map

Run on **Leaf3** (Leaf1/Leaf2 show `Total count : 2`):
```
admin@Leaf3:~$ show vxlan vlanvnimap
+---------+---------+-------+
| VTEP    | VLAN    |   VNI |
+=========+=========+=======+
| VTEP-10 | Vlan10  |  1010 |
+---------+---------+-------+
| VTEP-10 | Vlan20  |  2020 |
+---------+---------+-------+
| VTEP-10 | Vlan900 |  9900 |
+---------+---------+-------+
Total count : 3
```

#### VRF→VNI Map (L3VNI)

Run on **any leaf**:
```
admin@Leaf1:~$ show vxlan vrfvnimap
+---------+-------+-------+
| VTEP    | VRF   |   VNI |
+=========+=======+=======+
| VTEP-10 | Vrfa  |  9900 |
+---------+-------+-------+
Total count : 1
```
If `Vrfa` does not appear here, verify that **all three** of these steps were completed: `config vrf add Vrfa`, `config interface vrf bind Vlan900 Vrfa`, and `config vrf add_vrf_vni_map Vrfa 9900` (Step 3.3). The FRR-side `vrf Vrfa / vni 9900` (Step 4.2) is also required but does not populate this table — it tells BGP about the VNI.

#### Remote VTEPs (Learned via Type-3 IMET)

Run on **any leaf** — each should show the other two leaves as remote VTEPs:
```
admin@Leaf1:~$ show vxlan remotevtep
+---------+---------+-------------------+--------------+
| SIP     | DIP     | Creation Source   | OperStatus   |
+=========+=========+===================+==============+
| 1.1.1.1 | 2.2.2.2 | EVPN              | oper_up      |
+---------+---------+-------------------+--------------+
| 1.1.1.1 | 3.3.3.3 | EVPN              | oper_up      |
+---------+---------+-------------------+--------------+
Total count : 2
```
One entry per remote VTEP, all `oper_up`. Each entry was created from a received Type-3 IMET route. `oper_down` means the underlay tunnel to that VTEP is broken.

#### Remote MACs (Learned via Type-2)

Run on **Leaf1 and Leaf3** — the output differs depending on whether the leaf is multihomed or single-homed.

On **Leaf1** — only Leaf3's host MACs are remote (Leaf2's MACs are local via the shared ESI). The Vlan900 entry is Leaf3's Router MAC, used for L3VNI encapsulation:
```
admin@Leaf1:~$ show vxlan remotemac all
+---------+-------------------+----------------+-------+---------+
| VLAN    | MAC               | RemoteTunnel   |   VNI | Type    |
+=========+===================+================+=======+=========+
| Vlan10  | 2a:ff:f3:eb:89:6b | 3.3.3.3        |  1010 | dynamic |
+---------+-------------------+----------------+-------+---------+
| Vlan10  | ae:2c:d5:d2:40:cd | 3.3.3.3        |  1010 | dynamic |
+---------+-------------------+----------------+-------+---------+
| Vlan900 | 78:5a:34:3c:8c:00 | 3.3.3.3        |  9900 | dynamic |
+---------+-------------------+----------------+-------+---------+
Total count : 3
```

> MAC addresses are dynamically assigned and will differ in your lab. The important patterns are: Vlan10 entries point to 3.3.3.3 (Leaf3), and the Vlan900 entry is the remote leaf's Router MAC.

On **Leaf3** — Host12's MACs appear with two `RemoteTunnel` entries (one per multihomed leaf), confirming aliasing:
```
admin@Leaf3:~$ show vxlan remotemac all
+---------+-------------------+----------------+-------+---------+
| VLAN    | MAC               | RemoteTunnel   |   VNI | Type    |
+=========+===================+================+=======+=========+
| Vlan10  | 6e:98:c7:3a:24:1e | 1.1.1.1        |  1010 | dynamic |
|         |                   | 2.2.2.2        |       |         |
+---------+-------------------+----------------+-------+---------+
| Vlan10  | 72:dc:7e:4f:03:58 | 1.1.1.1        |  1010 | dynamic |
|         |                   | 2.2.2.2        |       |         |
+---------+-------------------+----------------+-------+---------+
| Vlan900 | 78:7e:c0:1e:74:00 | 2.2.2.2        |  9900 | dynamic |
+---------+-------------------+----------------+-------+---------+
| Vlan900 | 78:bc:08:17:88:00 | 1.1.1.1        |  9900 | dynamic |
+---------+-------------------+----------------+-------+---------+
Total count : 4
```

#### L2 Next-Hop Groups

Run on **Leaf3** — expect two individual VTEP NHGs and one ECMP group for the multihomed ESI:
```
admin@Leaf3:~$ show vxlan l2nexthopgroup
+-----------+-----------+---------------------+
|       NHG | Tunnels   | LocalMembers        |
+===========+===========+=====================+
| 268435458 | 2.2.2.2   |                     |
+-----------+-----------+---------------------+
| 268435459 | 1.1.1.1   |                     |
+-----------+-----------+---------------------+
| 536870913 |           | 268435459,268435458 |
+-----------+-----------+---------------------+
```

Run on **Leaf1** — only the peer leaf's NHG exists (Leaf2), since Leaf1 owns the MACs locally:
```
admin@Leaf1:~$ show vxlan l2nexthopgroup
+-----------+-----------+----------------+
|       NHG | Tunnels   | LocalMembers   |
+===========+===========+================+
| 268435458 | 2.2.2.2   |                |
+-----------+-----------+----------------+
| 536870913 |           | 268435458      |
+-----------+-----------+----------------+
```

---

### 5.2 EVPN Control Plane

Checks that the overlay BGP session to Spine4 is up, that all VNIs are correctly associated with the VRF, and that the EVPN MAC and route tables are populated with the expected entries from all peers.

#### EVPN BGP Session Summary

Run on **any leaf** — should show a single peer (Spine4's loopback `4.4.4.4`):
```
admin@Leaf3:~$ vtysh -c 'show bgp l2vpn evpn summary'

BGP router identifier 3.3.3.3, local AS number 3 vrf-id 0
BGP table version 0
RIB entries 17, using 3808 bytes of memory
Peers 1, using 22 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
4.4.4.4         4          4        32        32        0    0    0 00:03:36           14       24 N/A

Total number of neighbors 1
```
Single EVPN peer is the spine's loopback (`4.4.4.4`). `PfxRcd: 14` reflects all EVPN route types from Leaf1 and Leaf2 (Type-1 EAD, Type-2 MAC/IP, Type-3 IMET, Type-4 ES, Type-5 prefix).

#### VNI State

Run on **Leaf3** (it shows the most complete view with both L2 VNIs):
```
admin@Leaf3:~$ vtysh -c 'show evpn vni detail'
VNI: 2020
 Type: L2
 Domain: VTEP-10
 Tenant VRF: Vrfa
 VxLAN interface: VTEP-10-20
 SVI interface: Vlan20
 Local VTEP IP: 3.3.3.3
 Mcast group: 0.0.0.0
 No remote VTEPs known for this VNI
 Number of MACs (local and remote) known for this VNI: 2
 Number of ARPs (IPv4 and IPv6, local and remote) known for this VNI: 1

VNI: 1010
 Type: L2
 Domain: VTEP-10
 Tenant VRF: Vrfa
 VxLAN interface: VTEP-10-10
 SVI interface: Vlan10
 Local VTEP IP: 3.3.3.3
 Mcast group: 0.0.0.0
 Remote VTEPs for this VNI:
  1.1.1.1 flood: HER
  2.2.2.2 flood: HER
 Number of MACs (local and remote) known for this VNI: 4
 Number of ARPs (IPv4 and IPv6, local and remote) known for this VNI: 2

VNI: 9900
  Type: L3
  Tenant VRF: Vrfa
  Local Vtep Ip: 3.3.3.3
  Vxlan-Intf: VTEP-10-900
  SVI-If: Vlan900
  State: Up
  VNI Filter: none
  System MAC: 78:5a:34:3c:8c:00
  Router MAC: 78:5a:34:3c:8c:00
  L2 VNIs: 1010 2020
```
`Remote VTEPs` for VNI 1010 is the BUM flood list — one entry per remote leaf. VNI 2020 shows `No remote VTEPs` because only Leaf3 has Vlan20. L3VNI must show `State: Up` with a valid `Router MAC`. The `L2 VNIs` field lists all L2 VNIs associated with this VRF — on Leaf3 it shows both `1010` and `2020`.

#### MAC Table

Run on **Leaf3 and Leaf1** — the view differs between a single-homed and multihomed leaf.

On **Leaf3** — remote MACs point to the ESI (not a specific VTEP) because the source is multihomed. Leaf3 also shows local MACs for VNI 2020 (Host3b):
```
admin@Leaf3:~$ vtysh -c 'show evpn mac vni all'

VNI 2020 #MACs (local and remote) 2

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy, L=local
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
9e:8e:6d:3f:67:e3 local  L      Ethernet8                      20    0/0
5a:75:1d:f7:29:13 local  L      Ethernet8                      20    0/0

VNI 1010 #MACs (local and remote) 4

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy, L=local
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
6e:98:c7:3a:24:1e remote        03:00:34:34:34:34:34:00:00:01        0/0
2a:ff:f3:eb:89:6b local  L      Ethernet4                      10    0/0
72:dc:7e:4f:03:58 remote        03:00:34:34:34:34:34:00:00:01        0/0
ae:2c:d5:d2:40:cd local  L      Ethernet4                      10    0/0
```

On **Leaf1** — Host12's MACs are local on PortChannel1. The `PI` flag means peer-active/local-inactive (synced via ESI), and `NL` means sync-neighs/local:
```
admin@Leaf1:~$ vtysh -c 'show evpn mac vni all'

VNI 1010 #MACs (local and remote) 4

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy, L=local
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
6e:98:c7:3a:24:1e local  PI     PortChannel1                   10    0/0
2a:ff:f3:eb:89:6b remote        3.3.3.3                              0/0
72:dc:7e:4f:03:58 local  NL     PortChannel1                   10    0/0
ae:2c:d5:d2:40:cd remote        3.3.3.3                              0/0
```

> MAC addresses are dynamically assigned and will differ in your lab. The key things to verify are: (1) Leaf3's remote MACs point to the ESI `03:00:34:34:34:34:34:00:00:01` (not to a specific VTEP), (2) Leaf1's local MACs show PortChannel1 as the interface, and (3) Leaf3's VNI 2020 shows Host3b's MACs as local on Ethernet8.

#### Routes by Type

Run on any leaf to inspect individual route types:

```bash
vtysh -c 'show bgp l2vpn evpn route type ead'       # Type-1: EAD (multihoming)
vtysh -c 'show bgp l2vpn evpn route type macip'     # Type-2: MAC/IP
vtysh -c 'show bgp l2vpn evpn route type multicast' # Type-3: IMET (BUM flood list)
vtysh -c 'show bgp l2vpn evpn route type es'        # Type-4: ES (DF election)
vtysh -c 'show bgp l2vpn evpn route type prefix'    # Type-5: IP Prefix (L3)
```

---

### 5.3 Multihoming Verification

Confirms the Ethernet Segment is up, DF election has completed correctly (Leaf1 as DF, Leaf2 as non-DF), and that the LACP PortChannel is active. Leaf3 should see the ESI as fully remote.

#### Ethernet Segment Detail

Run on **Leaf1, Leaf2, and Leaf3** — the DF status will differ on each.

On **Leaf1** — DF winner:
```
admin@Leaf1:~$ vtysh -c 'show evpn es detail'
ESI: 03:00:34:34:34:34:34:00:00:01
 Type: Local,Remote
 Interface: PortChannel1
 State: up
 Bridge port: yes
 Ready for BGP: yes
 VNI Count: 1
 MAC Count: 2
 DF status: df 
 DF preference: 32767
 Nexthop group: 536870913
 VTEPs:
     2.2.2.2 df_alg: preference df_pref: 100 nh: 268435458
```

On **Leaf2** — non-DF:
```
admin@Leaf2:~$ vtysh -c 'show evpn es detail'
ESI: 03:00:34:34:34:34:34:00:00:01
 Type: Local,Remote
 Interface: PortChannel1
 State: up
 Bridge port: yes
 Ready for BGP: yes
 VNI Count: 1
 MAC Count: 2
 DF status: non-df 
 DF preference: 100
 Nexthop group: 536870913
 VTEPs:
     1.1.1.1 df_alg: preference df_pref: 32767 nh: 268435458
```

On **Leaf3** — ESI is fully remote:
```
admin@Leaf3:~$ vtysh -c 'show evpn es'
Type: B bypass, L local, R remote, N non-DF
ESI                            Type ES-IF                 VTEPs
03:00:34:34:34:34:34:00:00:01  R    -                     1.1.1.1,2.2.2.2
```

#### ES-EVI Binding

Run on **Leaf1 or Leaf2**:
```
admin@Leaf1:~$ vtysh -c 'show evpn es-evi'
Type: L local, R remote
VNI      ESI                            Type
1010     03:00:34:34:34:34:34:00:00:01  L
```
VNI 1010 is bound to the ESI. If this is empty, Type-1 EAD per-EVI routes will not be generated and remote leaves cannot set up aliasing.

#### PortChannel / LACP State

Run on **Leaf1 and Leaf2** — Leaf3 has no PortChannel:
```
admin@Leaf1:~$ show interfaces portchannel
Flags: A - active, I - inactive, Up - up, Dw - Down, N/A - not available,
       S - selected, D - deselected, * - not synced
  No.  Team Dev      Protocol     Ports
-----  ------------  -----------  ------------
    1  PortChannel1  LACP(A)(Up)  Ethernet4(S)
```
`LACP(A)(Up)` with `(S)` member confirms the bundle is active and the ESI is operational.

---

### 5.4 L3VNI / Symmetric IRB

Validates that Type-5 IP prefix routes are being imported into the VRF and that inter-subnet routing works across the fabric. Leaf3 provides the clearest view since it sees all remote prefixes via ECMP across both multihomed leaves.

#### VRF Route Table (FRR)

Run on **Leaf3** — as the single-homed leaf it shows the clearest view of remote Type-5 routes and ECMP:
```
admin@Leaf3:~$ vtysh -c 'show ip route vrf Vrfa'
VRF Vrfa:
C>* 10.0.1.0/24 is directly connected, Vlan10
B>* 10.0.1.10/32 [20/0] via 1.1.1.1, Vlan900 onlink, weight 1, RMAC 78:bc:08:17:88:00
  *                      via 2.2.2.2, Vlan900 onlink, weight 1, RMAC 78:7e:c0:1e:74:00
C>* 10.0.2.0/24 is directly connected, Vlan20
```
`C>*` entries are local connected subnets (Vlan10 and Vlan20). `B>*` entries are imported Type-2 host routes. `10.0.1.10/32` (Host12) has ECMP via both multihomed leaves — traffic to Host12 is load-balanced across the fabric. The `RMAC` field is the Router MAC of the remote leaf, used as the inner destination MAC when encapsulating routed VXLAN frames.

Run on **Leaf1** — shows remote routes for Leaf3's subnets:
```
admin@Leaf1:~$ vtysh -c 'show ip route vrf Vrfa'
VRF Vrfa:
C>* 10.0.1.0/24 is directly connected, Vlan10
B>* 10.0.1.30/32 [20/0] via 3.3.3.3, Vlan900 onlink, weight 1, RMAC 78:5a:34:3c:8c:00
B>* 10.0.2.0/24 [20/0] via 3.3.3.3, Vlan900 onlink, weight 1, RMAC 78:5a:34:3c:8c:00
B>* 10.0.2.40/32 [20/0] via 3.3.3.3, Vlan900 onlink, weight 1, RMAC 78:5a:34:3c:8c:00
```
Leaf1 sees `10.0.2.0/24` (Type-5 prefix route) and `10.0.2.40/32` (Type-2 host route) — both learned via Leaf3 (3.3.3.3). This confirms that placing Host3b's subnet in the same VRF allows automatic route distribution without any additional config on Leaf1 or Leaf2.

#### VRF BGP IPv4 Table

Run on **Leaf3**:
```
admin@Leaf3:~$ vtysh -c 'show bgp vrf Vrfa ipv4'

    Network          Next Hop            Metric LocPrf Weight Path
 *  10.0.1.0/24      1.1.1.1<                 0             0 1 ?
 *                   2.2.2.2<                 0             0 2 ?
 *>                  0.0.0.0                  0         32768 ?
 *= 10.0.1.10/32     1.1.1.1<                               0 1 i
 *>                  2.2.2.2<                               0 2 i
 *> 10.0.2.0/24      0.0.0.0                  0         32768 ?

Displayed  3 routes and 6 total paths
```

#### Type-5 Routes Detail

Run on **Leaf3**:
```
admin@Leaf3:~$ vtysh -c 'show bgp l2vpn evpn route type prefix'
Route Distinguisher: 1.1.1.1:4
 *> [5]:[0]:[24]:[10.0.1.0]
                    1.1.1.1                  0             0 1 ?
                    RT:9900:9900 ET:8 Rmac:78:bc:08:17:88:00
Route Distinguisher: 2.2.2.2:4
 *> [5]:[0]:[24]:[10.0.1.0]
                    2.2.2.2                  0             0 2 ?
                    RT:9900:9900 ET:8 Rmac:78:7e:c0:1e:74:00
Route Distinguisher: 3.3.3.3:3
 *> [5]:[0]:[24]:[10.0.1.0]
                    3.3.3.3                  0         32768 ?
                    ET:8 RT:9900:9900 Rmac:78:5a:34:3c:8c:00
 *> [5]:[0]:[24]:[10.0.2.0]
                    3.3.3.3                  0         32768 ?
                    ET:8 RT:9900:9900 Rmac:78:5a:34:3c:8c:00
```
Leaf3 originates two Type-5 prefixes — `10.0.1.0/24` (Vlan10) and `10.0.2.0/24` (Vlan20) — both with `RT:9900:9900`. Verify `RT:9900:9900` is present and consistent across all leaves — a mismatch in route-target prevents the prefix from being imported into the VRF on the receiving side.

---

### 5.5 Linux Kernel / Dataplane

Inspects the kernel-level data structures that back VXLAN forwarding — the VXLAN interfaces, the FDB (forwarding database), and the nexthop table. This is where you confirm the ECMP nexthop group for the multihomed ESI is correctly programmed in the kernel.

#### VXLAN Interfaces

Run on **Leaf3** (shows all three VXLAN interfaces including Vlan20):
```
admin@Leaf3:~$ ip link show type vxlan
77: VTEP-10-10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master Bridge state UNKNOWN
    link/ether 78:5a:34:3c:8c:00 brd ff:ff:ff:ff:ff:ff
79: VTEP-10-900: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master Bridge state UNKNOWN
    link/ether 78:5a:34:3c:8c:00 brd ff:ff:ff:ff:ff:ff
82: VTEP-10-20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master Bridge state UNKNOWN
    link/ether 78:5a:34:3c:8c:00 brd ff:ff:ff:ff:ff:ff
```
The interface name format is `<VTEP-name>-<vlan-id>`. All must be `UP`. The shared `link/ether` is the Router MAC. Leaf1/Leaf2 show only two VXLAN interfaces (VTEP-10-10 and VTEP-10-900) since they don't have Vlan20.

#### FDB (Forwarding Database)

Run on **Leaf3** — use the interface name from above to grep for relevant entries:
```
admin@Leaf3:~$ /sbin/bridge fdb | grep VTEP-10-10
72:dc:7e:4f:03:58 dev VTEP-10-10 vlan 10 extern_learn master Bridge remote
6e:98:c7:3a:24:1e dev VTEP-10-10 vlan 10 extern_learn master Bridge remote
00:00:00:00:00:00 dev VTEP-10-10 dst 2.2.2.2 self permanent
00:00:00:00:00:00 dev VTEP-10-10 dst 1.1.1.1 self permanent
72:dc:7e:4f:03:58 dev VTEP-10-10 nhid 536870913 self extern_learn remote
6e:98:c7:3a:24:1e dev VTEP-10-10 nhid 536870913 self extern_learn remote
```
All-zeros MAC entries (`dst 1.1.1.1`, `dst 2.2.2.2`) are the BUM flood list installed from Type-3 IMET routes. Remote host MAC entries point to `nhid 536870913` — the ECMP nexthop group for the multihomed ESI.

#### IP Nexthop Table

Run on **Leaf3** — expect one ECMP group combining both multihomed VTEPs:
```
admin@Leaf3:~$ ip nexthop list
id 268435458 via 2.2.2.2 scope link fdb
id 268435459 via 1.1.1.1 scope link fdb
id 536870913 group 268435459/268435458 fdb
```
The `group` entry is the ECMP nexthop for the multihomed ESI. Its ID matches `nhid 536870913` in the FDB. On Leaf1/Leaf2 only one individual NHG exists (pointing to the peer). You may also see additional kernel nexthops (e.g., `id 6`, `id 8`, `id 9`) — those are management-plane entries and can be ignored.

#### VRF Route Table (Kernel)

Run on **Leaf3**:
```
admin@Leaf3:~$ ip route show vrf Vrfa
10.0.1.0/24 dev Vlan10 proto kernel scope link src 10.0.1.1
10.0.1.10 proto bgp metric 20
	nexthop  encap ip id 9900 src 0.0.0.0 dst 1.1.1.1 ttl 0 tos 0 via 1.1.1.1 dev Vlan900 weight 1 onlink
	nexthop  encap ip id 9900 src 0.0.0.0 dst 2.2.2.2 ttl 0 tos 0 via 2.2.2.2 dev Vlan900 weight 1 onlink
10.0.2.0/24 dev Vlan20 proto kernel scope link src 10.0.2.1
```
`encap ip id 9900` shows VXLAN encapsulation using the L3VNI. The `10.0.1.10` host route has two ECMP nexthops — one via each multihomed leaf — confirming kernel-level load balancing for the multihomed host.

Run on **Leaf1**:
```
admin@Leaf1:~$ ip route show vrf Vrfa
10.0.1.0/24 dev Vlan10 proto kernel scope link src 10.0.1.1
10.0.1.30  encap ip id 9900 src 0.0.0.0 dst 3.3.3.3 ttl 0 tos 0 via 3.3.3.3 dev Vlan900 proto bgp metric 20 onlink
10.0.2.0/24  encap ip id 9900 src 0.0.0.0 dst 3.3.3.3 ttl 0 tos 0 via 3.3.3.3 dev Vlan900 proto bgp metric 20 onlink
10.0.2.40  encap ip id 9900 src 0.0.0.0 dst 3.3.3.3 ttl 0 tos 0 via 3.3.3.3 dev Vlan900 proto bgp metric 20 onlink
```
Leaf1 shows the `10.0.2.0/24` prefix route (Type-5) and the `10.0.2.40` host route (Type-2) — both via Leaf3's VTEP (3.3.3.3).

---

### 5.6 Platform NPU Verification

Confirms that the hardware forwarding tables on the ASIC match what FRR and the kernel have programmed. Useful for catching cases where the control plane looks healthy but traffic is not forwarding due to a SAI or ASIC programming issue.

#### Platform Summary

Run on **any leaf**:
```
admin@Leaf1:~$ sudo show platform summary
Platform: x86_64-8102_64h_o-r0
HwSKU: Cisco-8102-C64
ASIC: cisco-8000
ASIC Count: 1
Serial Number: CSNEJVU4J7B
Model Number: 8102-64H-O
Hardware Revision: 0.20
```

#### Hardware FIB (Route Table)

Run on **Leaf1** — remote loopbacks show as `next-hop`, local addresses as `for-us`, VRF routes as `vlan-next-hop`:
```
admin@Leaf1:~$ sudo show platform npu router route-table
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                                            Route Table                                                                            |
+-----------+------------------------------+---------------+----------+----------+---------------------------------+----------+--------------------+-------+--------+
| router-id | ip-prefix                    | dest-type     | dest-gid | dest-oid | dest-info                       | class-id | user-data          | drop  | ip-ver |
+-----------+------------------------------+---------------+----------+----------+---------------------------------+----------+--------------------+-------+--------+
| 0x0       | 1.1.1.1/32                   | for-us        | 0x0      | 0x0      | N/A                             | 0x0      | 36028797018963975  | False | 4      |
| 0x0       | 1.4.1.0/24                   | host          | 0x0      | 0x0      | N/A                             | 0x0      | 0                  | False | 4      |
| 0x0       | 1.4.1.1/32                   | for-us        | 0x0      | 0x0      | N/A                             | 0x0      | 36028797018963975  | False | 4      |
| 0x0       | 2.2.2.2/32                   | next-hop      | 0x0      | 0x0      | ['normal', '78:b1:eb:a1:74:00'] | 0x0      | 144115188075855873 | False | 4      |
| 0x0       | 2.4.1.0/24                   | next-hop      | 0x0      | 0x0      | ['normal', '78:b1:eb:a1:74:00'] | 0x0      | 144115188075855873 | False | 4      |
| 0x0       | 3.3.3.3/32                   | next-hop      | 0x0      | 0x0      | ['normal', '78:b1:eb:a1:74:00'] | 0x0      | 144115188075855873 | False | 4      |
| 0x0       | 3.4.1.0/24                   | next-hop      | 0x0      | 0x0      | ['normal', '78:b1:eb:a1:74:00'] | 0x0      | 144115188075855873 | False | 4      |
| 0x0       | 4.4.4.4/32                   | next-hop      | 0x0      | 0x0      | ['normal', '78:b1:eb:a1:74:00'] | 0x0      | 144115188075855873 | False | 4      |
| 0x0       | 10.0.1.0/24                  | host          | 0x0      | 0x0      | N/A                             | 0x0      | 0                  | False | 4      |
| 0x0       | 10.0.1.1/32                  | for-us        | 0x0      | 0x0      | N/A                             | 0x0      | 36028797018963975  | False | 4      |
| 0x0       | 10.0.2.0/24                  | vlan-next-hop | 0x0      | 0x0      | 74:6c:61:72:62:69               | 0x0      | 144115188075855874 | False | 4      |
| 0x0       | 10.0.2.40/32                 | vlan-next-hop | 0x0      | 0x0      | 74:6c:61:72:62:69               | 0x0      | 144115188075855874 | False | 4      |
+-----------+------------------------------+---------------+----------+----------+---------------------------------+----------+--------------------+-------+--------+
```
Key entries: `1.1.1.1/32` and `10.0.1.1/32` are `for-us` (local). Remote loopbacks (`2.2.2.2`, `3.3.3.3`, `4.4.4.4`) are `next-hop` via Spine4's MAC. VRF routes `10.0.2.0/24` and `10.0.2.40/32` are `vlan-next-hop` (VXLAN encapsulated via L3VNI).

#### ECMP Groups

Run on **Leaf3** — shows the ECMP groups for multihomed destinations with 2 `la_vxlan_next_hop` members each:
```
admin@Leaf3:~$ sudo show platform npu ecmp
Total ecmp group count: 3
 +----------------------------------------------------------------------------------------------------------------------------------------+
 |                                                                  ecmp                                                                  |
 +-----------------+---------------------+------------------+-------------+----------------------+----------------------------------------+
 | idx ecmp.member |       ecmp/oid      | ARS enabled/mode | member info |  l3_destination/gid  | member weight/no of duplicated members |
 +-----------------+---------------------+------------------+-------------+----------------------+----------------------------------------+
 |       0.0       | la_ecmp_group/10541 |       N/A        |     0/2     | la_vxlan_next_hop/-1 |                   1                    |
 |       0.1       | la_ecmp_group/10541 |       N/A        |     1/2     | la_vxlan_next_hop/-1 |                   1                    |
 |       2.0       | la_ecmp_group/10551 |       N/A        |     0/2     | la_vxlan_next_hop/-1 |                   1                    |
 |       2.1       | la_ecmp_group/10551 |       N/A        |     1/2     | la_vxlan_next_hop/-1 |                   1                    |
 +-----------------+---------------------+------------------+-------------+----------------------+----------------------------------------+
```
Each group has 2 members (one per multihomed VTEP — Leaf1 and Leaf2). This confirms the ASIC has programmed ECMP load-balancing for traffic toward the multihomed ESI.

#### Next-Hop Entries and Usage

Run on **Leaf1**:
```
admin@Leaf1:~$ sudo show platform npu next-hop entries
+-----------------------------------------------------------------------------------------------------------------------------------------+
|                                                             Next Hop Entries                                                            |
+-------+-----------+-------------+-------------------+----------------+-----------+---------+----------------+-----------+---------------+
| index | router-id | next-hop-id | mac-addr          | router-port-id | conn-type | conn-id | conn-user-data | ref-count | next-hop-type |
+-------+-----------+-------------+-------------------+----------------+-----------+---------+----------------+-----------+---------------+
| 0     | None      | 0xfff       | None              | None           | None      | None    | None           | 0         | DROP          |
| 1     | None      | 0x0         | None              | None           | None      | None    | None           | 3         | DROP          |
| 2     | 0x0       | 0x1         | 78:b1:eb:a1:74:00 | 0x404          | port      | 0x4     | 0000           | 8         | NORMAL        |
| 3     | 0x0       | 0x2         | 78:bc:08:17:88:00 | 0x405          | port      | 0x106   | None           | 0         | NORMAL        |
| 4     | 0x1       | 0x3         | 72:dc:7e:4f:03:58 | 0x40a          | None      | None    | None           | 0         | NORMAL        |
+-------+-----------+-------------+-------------------+----------------+-----------+---------+----------------+-----------+---------------+
```

```
admin@Leaf1:~$ sudo show platform npu next-hop usage
+-------------------------------+
|         Next Hop Usage        |
+---------------+---------------+
| num-next-hops | max-next-hops |
+---------------+---------------+
| 5             | 4096          |
+---------------+---------------+
```

#### PortChannel Hardware Programming

Run on **Leaf1 or Leaf2** (Leaf3 has no PortChannel):
```
admin@Leaf1:~$ sudo show platform npu lag members
################# show lag  #################
  LAG           |                     member info                    
                |   sysport macport           1st    1st             
oid  mode       |   oid     oid     slice ifg serdes lane speed state
----------------+----------------------------------------------------
10506 DYNAMIC    |   623     621     4     1   4      2308 100G  18(LINK_UP)
```
One LAG member (Ethernet4) in DYNAMIC (LACP) mode with state `LINK_UP`. This confirms the PortChannel is programmed in the ASIC and forwarding traffic.

---

## Appendix A — Underlay Configuration

> These steps are normally completed in Lab 2 (SONiC and FRR – Building a BGP Fabric). Only use this appendix if the underlay is not already configured when you start this lab (i.e., BGP neighbors are not established or loopback pings fail in Task 2).

### A.1 — Interface IP Addresses

Configure using the SONiC CLI on each device.

Run on **Leaf1**:
```bash
sudo config interface ip add Ethernet0 1.4.1.1/24
sudo config interface ip add Loopback0 1.1.1.1/32
sudo config save -y
```

Run on **Leaf2**:
```bash
sudo config interface ip add Ethernet0 2.4.1.2/24
sudo config interface ip add Loopback0 2.2.2.2/32
sudo config save -y
```

Run on **Leaf3**:
```bash
sudo config interface ip add Ethernet0 3.4.1.3/24
sudo config interface ip add Loopback0 3.3.3.3/32
sudo config save -y
```

Run on **Spine4**:
```bash
sudo config interface ip add Ethernet0 1.4.1.4/24
sudo config interface ip add Ethernet4 2.4.1.4/24
sudo config interface ip add Ethernet8 3.4.1.4/24
sudo config interface ip add Loopback0 4.4.4.4/32
sudo config save -y
```

### A.2 — Underlay BGP

Enter `vtysh` on each device and paste the corresponding block.

Run on **Leaf1**:
```
router bgp 1
 bgp router-id 1.1.1.1
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor 1.4.1.4 remote-as 4
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 1.4.1.4 activate
 exit-address-family
exit

route-map RM_SET_SRC permit 10
 set src 1.1.1.1
exit
route-map PASS permit 10
exit
ip protocol bgp route-map RM_SET_SRC
```

Run on **Leaf2**:
```
router bgp 2
 bgp router-id 2.2.2.2
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor 2.4.1.4 remote-as 4
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 2.4.1.4 activate
 exit-address-family
exit

route-map RM_SET_SRC permit 10
 set src 2.2.2.2
exit
route-map PASS permit 10
exit
ip protocol bgp route-map RM_SET_SRC
```

Run on **Leaf3**:
```
router bgp 3
 bgp router-id 3.3.3.3
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor 3.4.1.4 remote-as 4
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 3.4.1.4 activate
 exit-address-family
exit

route-map RM_SET_SRC permit 10
 set src 3.3.3.3
exit
route-map PASS permit 10
exit
ip protocol bgp route-map RM_SET_SRC
```

Run on **Spine4**:
```
router bgp 4
 bgp router-id 4.4.4.4
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor 1.4.1.1 remote-as 1
 neighbor 2.4.1.2 remote-as 2
 neighbor 3.4.1.3 remote-as 3
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 1.4.1.1 activate
  neighbor 2.4.1.2 activate
  neighbor 3.4.1.3 activate
 exit-address-family
exit
```

After applying, return to [Task 2 — Verify Underlay](#task-2--verify-underlay) to confirm BGP is established and loopbacks are reachable.


Proceed to [**Lab 6 (Building an AI Backend Network)**](../lab_6/lab_6-guide.md) to learn how we can implement and configure an AI Backend network fabric using SONiC.
