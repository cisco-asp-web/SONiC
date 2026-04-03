# SONiC VXLAN EVPN Lab Guide
## VXLAN DC Fabric with EVPN Multihoming

---

## Introduction

This lab walks you through building a VXLAN EVPN fabric on SONiC from scratch. You will configure a 3-leaf, 1-spine topology using eBGP as both the underlay and overlay control plane. The spine acts as an EVPN route server, reflecting routes between leaves without modifying BGP attributes.

Two leaves (Leaf1 and Leaf2) share a multihomed host via an active-active LACP PortChannel and a common Ethernet Segment Identifier (ESI). A third leaf (Leaf3) has a single-homed host. By the end of the lab you will have a working VXLAN fabric with L2 extension, L3 routing (symmetric IRB), and EVPN multihoming.

**SONiC version:** FRR 8.5.x, `split-unified` routing mode

---

## Table of Contents

1. [Lab Objectives](#lab-objectives)
2. [Topology & Reference](#topology--reference)
3. [Task 1 — Prerequisites](#task-1--prerequisites)
4. [Task 2 — Underlay Configuration](#task-2--underlay-configuration)
5. [Task 3 — SONiC Overlay Configuration](#task-3--sonic-overlay-configuration)
6. [Task 4 — FRR Overlay Configuration](#task-4--frr-overlay-configuration)
7. [Task 5 — Verification](#task-5--verification)
   - [5.1 Underlay Verification](#51-underlay-verification)
   - [5.2 VXLAN Tunnel State](#52-vxlan-tunnel-state)
   - [5.3 EVPN Control Plane](#53-evpn-control-plane)
   - [5.4 Multihoming](#54-multihoming-verification)
   - [5.5 L3VNI / Symmetric IRB](#55-l3vni--symmetric-irb)
   - [5.6 Linux Dataplane](#56-linux-kernel--dataplane)
   - [5.7 Platform NPU](#57-platform-npu-verification)

---

## Lab Objectives

By completing this lab you will be able to:

- [ ] Configure eBGP underlay between all leaves and the spine
- [ ] Verify loopback reachability across the fabric (required for VXLAN tunnels)
- [ ] Create VXLAN tunnel objects (VTEPs) and map VLANs to VNIs
- [ ] Configure a VRF with an L3VNI for symmetric IRB
- [ ] Configure a Static Anycast Gateway (SAG) with the same IP on all leaves
- [ ] Build a PortChannel on Leaf1 and Leaf2 with a shared ESI for multihoming
- [ ] Configure the EVPN overlay BGP session (loopback-to-loopback) to the route server
- [ ] Verify EVPN route types 1 through 5 are exchanged correctly
- [ ] Confirm ECMP nexthop groups are created for the multihomed ESI
- [ ] Verify L3 routing across the fabric via Type-5 prefixes

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
       └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
              │                    │                    │
              └────────ESI-1───────┘                    │
                   Multihomed Host                  Single-homed Host
```

### Device Addressing

| Device | AS | Loopback  | Spine4-facing IP |
|--------|----|-----------|------------------|
| Spine4 | 4  | 4.4.4.4   | —                |
| Leaf1  | 1  | 1.1.1.1   | 1.4.1.4/24       |
| Leaf2  | 2  | 2.2.2.2   | 2.4.1.4/24       |
| Leaf3  | 3  | 3.3.3.3   | 3.4.1.4/24       |

### VNI / VLAN / VRF Assignments

| Purpose   | VLAN    | VNI  | VRF  |
|-----------|---------|------|------|
| L2 tenant | Vlan10  | 1010 | Vrfa |
| L3VNI     | Vlan900 | 9900 | Vrfa |

### Tenant Network & SAG

| Parameter | Value                               |
|-----------|-------------------------------------|
| Subnet    | 10.0.1.0/24                         |
| SAG IP    | 10.0.1.1/24 (same on all leaves)    |
| SAG MAC   | 0a:aa:aa:aa:aa:aa (same on all leaves) |

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

## Task 2 — Underlay Configuration

> **Goal:** Establish eBGP sessions between each leaf and the spine over the direct point-to-point link. Each leaf advertises its loopback and connected routes. The spine redistributes all loopbacks back to all leaves. Loopback reachability is required before VXLAN tunnels can form.

### Step 2.1 — Interface IP Addresses

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

### Step 2.2 — Underlay BGP

Enter `vtysh` on each device (`sudo vtysh`) and paste the corresponding block.

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

### Step 2.3 — Verify Underlay

Run on any leaf. All remote loopbacks should be reachable before proceeding.

```bash
# Check BGP session is Established and prefix count > 0
sudo vtysh -c 'show bgp ipv4 unicast summary'

# Ping all remote VTEP loopbacks — must be 0% loss
ping 2.2.2.2 -c3
ping 3.3.3.3 -c3
ping 4.4.4.4 -c3
```

> **Do not proceed to Task 3 until all loopback pings succeed.**

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
```

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
sudo config save -y
```

Run on **Leaf2**:
```bash
sudo config portchannel add PortChannel1
sudo config portchannel member add PortChannel1 Ethernet4
sudo config vlan member add -u 10 PortChannel1
sudo config save -y
```

#### Leaf3 — Single-Homed Host via Access Port

Run on **Leaf3**:
```bash
sudo config vlan member add -u 10 Ethernet4
sudo config save -y
```

---

## Task 4 — FRR Overlay Configuration

> **Goal:** Configure the EVPN overlay BGP session from each leaf to the spine's loopback (as EVPN route server), enable EVPN MH global settings, bind the VRF VNI in FRR, configure the ESI on the PortChannels (Leaf1/Leaf2 only), and configure the spine as route server.

### Step 4.1 — Global EVPN MH Setting

Run on **Leaf1, Leaf2, and Leaf3** — enter `sudo vtysh` first, then paste:
```
evpn mh redirect-off
```

### Step 4.2 — VRF VNI Binding in FRR

Run on **Leaf1, Leaf2, and Leaf3** — in `sudo vtysh`:
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
  advertise-all-vni
  no use-es-l3nhg
  vni 1010
   route-target import 1010:1010
   route-target export 1010:1010
  exit-vni
 exit-address-family
exit
```

Run on **Leaf3**:
```
router bgp 3
 neighbor 4.4.4.4 remote-as 4
 neighbor 4.4.4.4 ebgp-multihop 10
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family l2vpn evpn
  neighbor 4.4.4.4 activate
  advertise-all-vni
  no use-es-l3nhg
  vni 1010
   route-target import 1010:1010
   route-target export 1010:1010
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

## Task 5 — Verification

### 5.1 Underlay Verification

#### BGP Neighbor State
```
admin@Leaf1:~$ sudo vtysh -c 'show bgp ipv4 unicast summary'

BGP router identifier 1.1.1.1, local AS number 1 vrf-id 0
BGP table version 127
RIB entries 9, using 2016 bytes of memory
Peers 2, using 44 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
1.4.1.4         4          4     21709     21698        0    0    0 01w6d07h            4        5 SPINE4

Total number of neighbors 2
```
The underlay peer (`1.4.1.4`) is Spine4's Ethernet0 IP. `State/PfxRcd` of `4` means four loopback host routes are received. If this shows `Idle` or `Active`, check that interface IPs are configured and the cable is connected.

#### Loopback Reachability
```
admin@Leaf1:~$ ping 3.3.3.3 -c3
PING 3.3.3.3 (3.3.3.3) 56(84) bytes of data.
64 bytes from 3.3.3.3: icmp_seq=1 ttl=63 time=0.4 ms
64 bytes from 3.3.3.3: icmp_seq=2 ttl=63 time=0.4 ms
64 bytes from 3.3.3.3: icmp_seq=3 ttl=63 time=0.4 ms

--- 3.3.3.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```
`ttl=63` confirms single-hop transit through the spine. 0% packet loss is required before VXLAN tunnels can form.

---

### 5.2 VXLAN Tunnel State

#### VTEP Source Interface
```
admin@Leaf1:~$ show vxlan interface
VTEP Information:

	VTEP Name : VTEP-10, SIP  : 1.1.1.1
	NVO Name  : NVO,  VTEP : VTEP-10
	Source interface  : Loopback0
```

#### Tunnel and VNI Mappings
```
admin@Leaf1:~$ show vxlan tunnel
vxlan tunnel name    source ip    destination ip    tunnel map name    tunnel map mapping(vni -> vlan)
-------------------  -----------  ----------------  -----------------  ---------------------------------
VTEP-10              1.1.1.1                        map_1010_Vlan10    1010 -> Vlan10
                                                    map_9900_Vlan900   9900 -> Vlan900
```

#### VLAN→VNI Map
```
admin@Leaf1:~$ show vxlan vlanvnimap
+---------+---------+-------+
| VTEP    | VLAN    |   VNI |
+=========+=========+=======+
| VTEP-10 | Vlan10  |  1010 |
+---------+---------+-------+
| VTEP-10 | Vlan900 |  9900 |
+---------+---------+-------+
Total count : 2
```

#### VRF→VNI Map (L3VNI)
```
admin@Leaf1:~$ show vxlan vrfvnimap
+---------+-------+-------+
| VTEP    | VRF   |   VNI |
+=========+=======+=======+
| VTEP-10 | Vrfa  |  9900 |
+---------+-------+-------+
Total count : 1
```
If `Vrfa` does not appear here, the `config interface vrf bind Vlan900 Vrfa` step was missed or the FRR `vrf Vrfa / vni 9900` binding is missing.

#### Remote VTEPs (Learned via Type-3 IMET)
```
admin@Leaf1:~$ show vxlan remotevtep
+---------+---------+-------------------+--------------+
| SIP     | DIP     | Creation Source   | OperStatus   |
+=========+=========+===================+==============+
| 1.1.1.1 | 2.2.2.2 | EVPN              | oper_up      |
+---------+---------+-------------------+--------------+
| 1.1.1.1 | 3.3.3.3 | EVPN              | oper_up      |
+---------+---------+-------------------+--------------+
| 1.1.1.1 | 5.5.5.5 | EVPN              | oper_up      |
+---------+---------+-------------------+--------------+
Total count : 3
```
One entry per remote VTEP, all `oper_up`. Each entry was created from a received Type-3 IMET route. `oper_down` means the underlay tunnel to that VTEP is broken.

#### Remote MACs (Learned via Type-2)

From **Leaf1** — only Leaf3's host MAC is remote (Leaf2's MACs are local via the shared ESI):
```
admin@Leaf1:~$ show vxlan remotemac all
+---------+-------------------+----------------+-------+---------+
| VLAN    | MAC               | RemoteTunnel   |   VNI | Type    |
+=========+===================+================+=======+=========+
| Vlan10  | 00:08:9e:21:4c:00 | 3.3.3.3        |  1010 | dynamic |
+---------+-------------------+----------------+-------+---------+
| Vlan900 | 02:11:90:38:f4:4c | 5.5.5.5        |  9900 | dynamic |
+---------+-------------------+----------------+-------+---------+
| Vlan900 | 78:cb:d6:19:50:00 | 3.3.3.3        |  9900 | dynamic |
+---------+-------------------+----------------+-------+---------+
Total count : 3
```

From **Leaf3** — Host A's MACs appear with two `RemoteTunnel` entries (one per multihomed leaf), confirming aliasing:
```
admin@Leaf3:~$ show vxlan remotemac all
+---------+-------------------+----------------+-------+---------+
| VLAN    | MAC               | RemoteTunnel   |   VNI | Type    |
+=========+===================+================+=======+=========+
| Vlan10  | 00:9e:20:a5:e0:00 | 1.1.1.1        |  1010 | dynamic |
|         |                   | 2.2.2.2        |       |         |
+---------+-------------------+----------------+-------+---------+
| Vlan10  | 00:9e:20:a5:e0:08 | 1.1.1.1        |  1010 | dynamic |
|         |                   | 2.2.2.2        |       |         |
+---------+-------------------+----------------+-------+---------+
| Vlan10  | 00:9e:20:a5:e1:04 | 1.1.1.1        |  1010 | dynamic |
|         |                   | 2.2.2.2        |       |         |
+---------+-------------------+----------------+-------+---------+
...
Total count : 7
```

#### L2 Next-Hop Groups

From **Leaf3** — two individual VTEP NHGs and one ECMP group for the multihomed ESI:
```
admin@Leaf3:~$ show vxlan l2nexthopgroup
+-----------+-----------+---------------------+
|       NHG | Tunnels   | LocalMembers        |
+===========+===========+=====================+
| 268435458 | 1.1.1.1   |                     |
+-----------+-----------+---------------------+
| 268435459 | 2.2.2.2   |                     |
+-----------+-----------+---------------------+
| 536870913 |           | 268435458,268435459 |
+-----------+-----------+---------------------+
```

From **Leaf1** — only the peer leaf's NHG exists (Leaf2), since Leaf1 owns the MACs locally:
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

### 5.3 EVPN Control Plane

#### EVPN BGP Session Summary
```
admin@Leaf3:~$ sudo vtysh -c 'show bgp l2vpn evpn summary'

BGP router identifier 3.3.3.3, local AS number 3 vrf-id 0
RIB entries 21, using 4704 bytes of memory
Peers 1, using 22 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
4.4.4.4         4          4     22088     21860        0    0    0 01w5d09h           16       20 N/A

Total number of neighbors 1
```
Single EVPN peer is the spine's loopback (`4.4.4.4`). `PfxRcd: 16` reflects all EVPN route types from Leaf1 and Leaf2.

#### VNI State
```
admin@Leaf3:~$ sudo vtysh -c 'show evpn vni detail'
VNI: 1010
 Type: L2
 Domain: VTEP-10
 Tenant VRF: Vrfa
 VxLAN interface: VTEP-10-10
 Local VTEP IP: 3.3.3.3
 Remote VTEPs for this VNI:
  2.2.2.2 flood: HER
  1.1.1.1 flood: HER
 Number of MACs (local and remote) known for this VNI: 4
 Number of ARPs (IPv4 and IPv6, local and remote) known for this VNI: 2

VNI: 9900
  Type: L3
  Tenant VRF: Vrfa
  Local Vtep Ip: 3.3.3.3
  State: Up
  System MAC: 78:cb:d6:19:50:00
  Router MAC: 78:cb:d6:19:50:00
  L2 VNIs: 1010
```
`Remote VTEPs` for VNI 1010 is the BUM flood list — one entry per remote leaf. L3VNI must show `State: Up` with a valid `Router MAC`.

#### MAC Table

From **Leaf3** — remote MACs point to the ESI (not a specific VTEP) because the source is multihomed:
```
admin@Leaf3:~$ sudo vtysh -c 'show evpn mac vni all'

VNI 1010 #MACs (local and remote) 4

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy, L=local
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
00:9e:20:a5:e0:08 remote        03:00:34:34:34:34:34:00:00:01        0/0
00:08:9e:21:4c:00 local  L      Ethernet4                      10    0/0
00:9e:20:a5:e1:04 remote        03:00:34:34:34:34:34:00:00:01        0/0
00:9e:20:a5:e0:00 remote        03:00:34:34:34:34:34:00:00:01        0/0
```

From **Leaf1** — Host A's MACs are all local (PortChannel1):
```
admin@Leaf1:~$ sudo vtysh -c 'show evpn mac vni all'

VNI 1010 #MACs (local and remote) 4

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy, L=local
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
00:9e:20:a5:e0:08 local  PI     PortChannel1                   10    0/0
00:08:9e:21:4c:00 remote        3.3.3.3                              0/0
00:9e:20:a5:e1:04 local  NPI    PortChannel1                   10    0/0
00:9e:20:a5:e0:00 local  L      PortChannel1                   10    0/0
```

#### Routes by Type

Run on any leaf to inspect individual route types:

```bash
sudo vtysh -c 'show bgp l2vpn evpn route type ead'       # Type-1: EAD (multihoming)
sudo vtysh -c 'show bgp l2vpn evpn route type macip'     # Type-2: MAC/IP
sudo vtysh -c 'show bgp l2vpn evpn route type multicast' # Type-3: IMET (BUM flood list)
sudo vtysh -c 'show bgp l2vpn evpn route type es'        # Type-4: ES (DF election)
sudo vtysh -c 'show bgp l2vpn evpn route type prefix'    # Type-5: IP Prefix (L3)
```

---

### 5.4 Multihoming Verification

#### Ethernet Segment Detail

From **Leaf1** — DF winner:
```
admin@Leaf1:~$ sudo vtysh -c 'show evpn es detail'
ESI: 03:00:34:34:34:34:34:00:00:01
 Type: Local,Remote
 Interface: PortChannel1
 State: up
 Bridge port: yes
 Ready for BGP: yes
 VNI Count: 1
 MAC Count: 3
 DF status: df
 DF preference: 32767
 Nexthop group: 536870913
 VTEPs:
     2.2.2.2 df_alg: preference df_pref: 100 nh: 268435458
```

From **Leaf2** — non-DF:
```
admin@Leaf2:~$ sudo vtysh -c 'show evpn es detail'
ESI: 03:00:34:34:34:34:34:00:00:01
 Type: Local,Remote
 Interface: PortChannel1
 State: up
 Bridge port: yes
 Ready for BGP: yes
 VNI Count: 1
 MAC Count: 3
 DF status: non-df
 DF preference: 100
 Nexthop group: 536870913
 VTEPs:
     1.1.1.1 df_alg: preference df_pref: 32767 nh: 268435458
```

From **Leaf3** — ESI is fully remote:
```
admin@Leaf3:~$ sudo vtysh -c 'show evpn es'
Type: B bypass, L local, R remote, N non-DF
ESI                            Type ES-IF                 VTEPs
03:00:34:34:34:34:34:00:00:01  R    -                     1.1.1.1,2.2.2.2
```

#### ES-EVI Binding
```
admin@Leaf1:~$ sudo vtysh -c 'show evpn es-evi'
Type: L local, R remote
VNI      ESI                            Type
1010     03:00:34:34:34:34:34:00:00:01  L
```
VNI 1010 is bound to the ESI. If this is empty, Type-1 EAD per-EVI routes will not be generated and remote leaves cannot set up aliasing.

#### PortChannel / LACP State
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

### 5.5 L3VNI / Symmetric IRB

#### VRF Route Table (FRR)
```
admin@Leaf3:~$ sudo vtysh -c 'show ip route vrf Vrfa'
VRF Vrfa:
B>* 8.21.0.0/24 [20/0] via 5.5.5.5, Vlan900 onlink, weight 1, RMAC 02:11:90:38:f4:4c
C>* 10.0.1.0/24 is directly connected, Vlan10
B>* 10.0.1.1/32 [20/0] via 1.1.1.1, Vlan900 onlink, weight 1, RMAC 78:75:2d:22:a0:00
  *                     via 2.2.2.2, Vlan900 onlink, weight 1, RMAC 78:07:85:d7:44:00
```
`C>*` is the local connected subnet. `B>*` entries are imported Type-5 routes. `10.0.1.1/32` has ECMP via both multihomed leaves. The `RMAC` field is the Router MAC of the remote leaf — it is used as the inner destination MAC when encapsulating routed VXLAN frames.

#### VRF BGP IPv4 Table
```
admin@Leaf3:~$ sudo vtysh -c 'show bgp vrf Vrfa ipv4'

    Network          Next Hop            Metric LocPrf Weight Path
 *> 8.21.0.0/24      5.5.5.5<                               0 5 ?
 *  10.0.1.0/24      2.4.1.2<                 0             0 2 ?
 *                   1.4.1.1<                 0             0 1 ?
 *>                  0.0.0.0                  0         32768 ?
 *= 10.0.1.1/32      2.2.2.2<                               0 2 i
 *>                  1.1.1.1<                               0 1 i

Displayed  3 routes and 6 total paths
```

#### Type-5 Routes Detail
```
admin@Leaf3:~$ sudo vtysh -c 'show bgp l2vpn evpn route type prefix'
Route Distinguisher: 1.1.1.1:3
 *> [5]:[0]:[24]:[10.0.1.0]
                    1.1.1.1                  0             0 1 ?
                    RT:9900:9900 ET:8 Rmac:78:75:2d:22:a0:00
Route Distinguisher: 2.2.2.2:4
 *> [5]:[0]:[24]:[10.0.1.0]
                    2.2.2.2                  0             0 2 ?
                    RT:9900:9900 ET:8 Rmac:78:07:85:d7:44:00
Route Distinguisher: 3.3.3.3:4
 *> [5]:[0]:[24]:[10.0.1.0]
                    3.3.3.3                  0         32768 ?
                    ET:8 RT:9900:9900 Rmac:78:cb:d6:19:50:00
Route Distinguisher: 5.5.5.5:0
 *> [5]:[0]:[24]:[8.21.0.0]
                    5.5.5.5                                0 5 ?
                    RT:9900:9900 ET:8 Rmac:02:11:90:38:f4:4c
```
Verify `RT:9900:9900` is present and consistent across all leaves — a mismatch in route-target prevents the prefix from being imported into the VRF on the receiving side.

---

### 5.6 Linux Kernel / Dataplane

#### VXLAN Interfaces
```
admin@Leaf3:~$ ip link show type vxlan
81: VTEP-10-10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master Bridge state UNKNOWN
    link/ether 78:cb:d6:19:50:00 brd ff:ff:ff:ff:ff:ff
82: VTEP-10-900: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master Bridge state UNKNOWN
    link/ether 78:cb:d6:19:50:00 brd ff:ff:ff:ff:ff:ff
```
The interface name format is `<VTEP-name>-<vlan-id>`. Both must be `UP`. The shared `link/ether` is the Router MAC.

#### FDB (Forwarding Database)

Use the interface name from above to grep for relevant entries:
```
admin@Leaf3:~$ /sbin/bridge fdb | grep VTEP-10-10
00:9e:20:a5:e0:08 dev VTEP-10-10 vlan 10 extern_learn master Bridge remote
00:9e:20:a5:e0:00 dev VTEP-10-10 vlan 10 extern_learn master Bridge remote
00:9e:20:a5:e1:04 dev VTEP-10-10 vlan 10 extern_learn master Bridge remote
00:00:00:00:00:00 dev VTEP-10-10 dst 1.1.1.1 self permanent
00:00:00:00:00:00 dev VTEP-10-10 dst 2.2.2.2 self permanent
00:9e:20:a5:e0:00 dev VTEP-10-10 nhid 536870913 self extern_learn remote
00:9e:20:a5:e1:04 dev VTEP-10-10 nhid 536870913 self extern_learn remote
00:9e:20:a5:e0:08 dev VTEP-10-10 nhid 536870913 self extern_learn remote
```
All-zeros MAC entries (`dst 1.1.1.1`, `dst 2.2.2.2`) are the BUM flood list installed from Type-3 IMET routes. Remote host MAC entries point to `nhid 536870913` — the ECMP nexthop group for the multihomed ESI.

#### IP Nexthop Table
```
admin@Leaf3:~$ ip nexthop list
id 268435458 via 1.1.1.1 scope link fdb
id 268435459 via 2.2.2.2 scope link fdb
id 536870913 group 268435458/268435459 fdb
```
The `group` entry is the ECMP nexthop for the multihomed ESI. Its ID matches `nhid 536870913` in the FDB. On Leaf1/Leaf2 only one individual NHG exists (pointing to the peer).

#### VRF Route Table (Kernel)
```
admin@Leaf3:~$ ip route show vrf Vrfa
8.21.0.0/24  encap ip id 9900 src 0.0.0.0 dst 5.5.5.5 ttl 0 tos 0 via 5.5.5.5 dev Vlan900 proto bgp metric 20 onlink
10.0.1.0/24  dev Vlan10 proto kernel scope link src 10.0.1.1
10.0.1.2     encap ip id 9900 src 0.0.0.0 dst 3.3.3.3 ttl 0 tos 0 via 3.3.3.3 dev Vlan900 proto bgp metric 20 onlink
```
`encap ip id 9900` shows VXLAN encapsulation using the L3VNI. `dst X.X.X.X` is the remote VTEP the packet is tunneled to.

---

### 5.7 Platform NPU Verification

Run on any leaf to confirm hardware programming:

```bash
# Hardware FIB — remote loopbacks show as next-hop, local addresses as for-us/host
sudo show platform npu router route-table

# ECMP groups — expect one group with 2 la_vxlan_next_hop members for the multihomed ESI
sudo show platform npu ecmp

# Next-hop adjacency table
sudo show platform npu next-hop entries
sudo show platform npu next-hop usage

# PortChannel hardware programming — Leaf1/Leaf2 only
sudo show platform npu lag members

# Confirm platform and HWSKU
sudo show platform summary
```
