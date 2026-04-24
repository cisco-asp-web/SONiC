# Deep-Dive Exercise — Tracking a Route Through the Entire SONiC Pipeline

This exercise adds a new loopback on **Leaf1** and tracks the prefix as it propagates through
every layer of the SONiC stack — from `CONFIG_DB` all the way to the Silicon One ASIC and
out to **Spine4** via BGP. By the end you will have watched a single route touch every
significant component in the SONiC control plane.

---

## Objective

- Understand the end-to-end SONiC control plane pipeline hands-on
- Observe CONFIG_DB → APP_DB → ASIC_DB → hardware → BGP in real time
- Understand why route policy must be updated before a prefix is advertised

---

## Part 1 — Prepare Your Terminals

Before adding anything, open **5 terminals on Leaf1** and **1 terminal on Spine4** side by side.
Each terminal watches one layer of the stack.

**Terminal 1 — Leaf1: CONFIG_DB**
```bash
watch -n 1 "sonic-db-cli CONFIG_DB hgetall 'INTERFACE|Loopback1'"
```

**Terminal 2 — Leaf1: APP_DB**
```bash
watch -n 1 "sonic-db-cli APP_DB hgetall 'ROUTE_TABLE:10.10.10.10/32'"
```

**Terminal 3 — Leaf1: ASIC_DB**
```bash
watch -n 1 "sonic-db-cli ASIC_DB keys 'ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY*' | tr ',' '\n' | grep 10.10.10.10"
```

**Terminal 4 — Leaf1: Linux kernel FIB**
```bash
watch -n 1 "ip route show 10.10.10.10/32"
```

**Terminal 5 — Leaf1: FRR RIB**
```bash
watch -n 1 "sudo vtysh -c 'show ip route 10.10.10.10/32'"
```

**Terminal 6 — Spine4: BGP RIB**
```bash
watch -n 1 "sudo vtysh -c 'show bgp ipv4 unicast 10.10.10.10/32'"
```

> 💡 All terminals should currently show empty output or "not found". This is your baseline.
> Do not proceed until all watches are running.

---

## Part 2 — Add the New Loopback

On **Leaf1** (in a new terminal, not one of the watch terminals):

```bash
sudo config interface ip add Loopback1 10.10.10.10/32
```

Watch your terminals immediately after. You will see the pipeline light up layer by layer
within seconds.

---

## Part 3 — Observe Each Layer

### Terminal 1 — CONFIG_DB (instant)

The SONiC `config` CLI writes to CONFIG_DB first, before anything else happens.

```bash
sonic-db-cli CONFIG_DB hgetall 'INTERFACE|Loopback1'
# {'NULL': 'NULL'}

sonic-db-cli CONFIG_DB hgetall 'INTERFACE|Loopback1|10.10.10.10/32'
# {'scope': 'global', 'family': 'IPv4'}
```

`intfmgrd` reads this change from CONFIG_DB and creates the Linux interface, then notifies
zebra of the new connected route.

---

### Terminal 2 — APP_DB (within 1–2 seconds)

`fpmsyncd` receives the route from zebra via the FPM (Forwarding Plane Manager) socket and
writes it into APP_DB. This is the handoff point between FRR and the SONiC database world.

```bash
sonic-db-cli APP_DB hgetall 'ROUTE_TABLE:10.10.10.10/32'
# {'nexthop': '0.0.0.0', 'ifname': 'Loopback1'}
```

`nexthop: 0.0.0.0` is correct for a connected/local route — it means the prefix is directly
attached, no next-hop needed.

---

### Terminal 3 — ASIC_DB (within 1–2 seconds of APP_DB)

`orchagent` reads the new entry from APP_DB, translates it into a SAI route object, and writes
it to ASIC_DB. `syncd` then programs the ASIC via the SAI API.

```bash
sonic-db-cli ASIC_DB keys 'ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY*' | tr ',' '\n' | grep 10.10.10.10
```

Then inspect the full SAI object:

```bash
sonic-db-cli ASIC_DB hgetall 'ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{"dest":"10.10.10.10/32","switch_id":"oid:0x21000000000000","vr_id":"oid:0x3000000000042"}'
```

---

### Terminal 4 — Linux kernel FIB (near-instant)

Zebra installs the route into the kernel simultaneously via Netlink — independently of the
APP_DB/ASIC_DB pipeline.

```bash
ip route show 10.10.10.10/32
# 10.10.10.10 dev Loopback1 proto kernel scope host src 10.10.10.10
```

`proto kernel` means the kernel itself owns this route (it is a local/connected route).
Compare this with BGP-learned routes which show `proto bgp`.

---

### Terminal 5 — FRR RIB

Zebra has added the connected route to the FRR RIB immediately after interface creation.

```bash
sudo vtysh -c 'show ip route 10.10.10.10/32'
# C>* 10.10.10.10/32 is directly connected, Loopback1
```

`C>*` = connected, selected, installed in FIB. But notice — this does **not** automatically
mean the prefix will be advertised to Spine4. That depends entirely on BGP policy.

---

### Terminal 6 — Spine4 BGP RIB (nothing yet — expected)

```bash
sudo vtysh -c 'show bgp ipv4 unicast 10.10.10.10/32'
# % Network not in table
```

The prefix is not advertised to Spine4 yet. The current `ADVERTISE` route-map on Leaf1 only
permits prefixes matching `prefix-list LOOPBACKS`, which currently only contains `1.1.1.1/32`.
This is intentional — you must explicitly update the policy before the prefix is shared.

---

## Part 4 — Update the Route Policy

This is the critical step. Without it, BGP will never advertise the new prefix regardless of
what exists in the routing table. FRR's `bgp ebgp-requires-policy` enforcement means
outbound advertisements are silently dropped if they do not match an outbound permit.

Enter `vtysh` on **Leaf1** and update the prefix-list to include the new loopback:

```bash
sudo vtysh
```

```
pod9-leaf1# configure terminal
pod9-leaf1(config)# ip prefix-list LOOPBACKS seq 10 permit 10.10.10.10/32
pod9-leaf1(config)# end
pod9-leaf1# write memory
```

> ⚠️ Adding the prefix to `prefix-list LOOPBACKS` is sufficient — the existing `ADVERTISE`
> route-map already references this prefix-list. No changes to the route-map itself are needed.

Alternatively, if you want to advertise **all** loopbacks without updating the prefix-list
each time, you could match on a broader prefix:

```
ip prefix-list LOOPBACKS seq 10 permit 10.0.0.0/8 le 32
```

---

## Part 5 — Observe BGP Advertisement

After updating the prefix-list, BGP will detect the policy change and re-evaluate outbound
advertisements on the next update cycle. You can force it immediately:

```bash
# Soft reset the outbound policy toward Spine4 (no session reset)
sudo vtysh -c 'clear bgp ipv4 unicast 1.4.1.4 soft out'
```

Now watch **Terminal 6 on Spine4**:

```bash
sudo vtysh -c 'show bgp ipv4 unicast 10.10.10.10/32'
```

You should now see:
```
BGP routing table entry for 10.10.10.10/32
Paths: (1 available, best #1, table default)
  1.4.1.1 from 1.4.1.1 (1.1.1.1)
    Origin incomplete, metric 0, localpref 100, valid, external, best (First path received)
    Last update: 0:00:xx ago
```

Also verify on Spine4's FRR RIB and Linux kernel:

```bash
# FRR RIB on Spine4
sudo vtysh -c 'show ip route 10.10.10.10/32'

# Linux kernel FIB on Spine4
ip route show proto bgp | grep 10.10.10.10
```

---

## Part 6 — Verify End-to-End Reachability

From **Spine4**, ping the new loopback using Spine4's own loopback as the source:

```bash
ping 10.10.10.10 -c 3 -I 4.4.4.4
```

From **Leaf2** or **Leaf3** (if they also have routes to `10.10.10.10/32` via Spine4):

```bash
ping 10.10.10.10 -c 3 -I Loopback0
```

---

## Part 7 — The Full Pipeline Visualised

```
sudo config interface ip add Loopback1 10.10.10.10/32
          │
          ▼
     CONFIG_DB                ← written immediately by config CLI
          │
          ▼
      intfmgrd                ← reads CONFIG_DB, creates Linux interface
          │
          ▼
    zebra / FRR               ← detects new connected route
          │
          ├──────────────────────────────────────────────┐
          ▼                                              ▼
  Linux kernel FIB       fpmsyncd ──► APP_DB            │
  (via Netlink)               (ROUTE_TABLE)              │
  proto kernel                     │                     │
                                   ▼                     │
                             orchagent                   │
                                   │                     │
                                   ▼                     │
                              ASIC_DB                    │
                         (SAI route object)              │
                                   │                     │
                                   ▼                     │
                            syncd / SAI                  │
                                   │                     │
                                   ▼                     │
                           Silicon One ASIC              │
                        (prefix in hardware FIB)         │
                                                         │
                                   ┌─────────────────────┘
                                   ▼
                              bgpd (FRR)
                       evaluates outbound policy
                      (ADVERTISE route-map + prefix-list LOOPBACKS)
                                   │
                          ┌────────┴────────┐
                          │ prefix matches? │
                    NO ───┴─── YES ─────────┼──► UPDATE sent to Spine4
                    (silent drop)           │
                                            ▼
                                     Spine4 BGP RIB
                                            │
                                            ▼
                                   Spine4 Linux kernel FIB
                                     (proto bgp)
```

---

## Bonus Challenges

> **Challenge 1:** After adding the new loopback, check `vtysh -c 'show route-map'` on Leaf1.
> What changed in the `ADVERTISE` invocation counters compared to before? Does the number
> of matches increase? What does that tell you about when BGP evaluates outbound policy?

> **Challenge 2:** Remove the loopback and watch the withdrawal propagate:
> ```bash
> sudo config interface ip remove Loopback1 10.10.10.10/32
> ```
> In what order do the layers clear? Is it the reverse of the add sequence?

> **Challenge 3:** Query `STATE_DB` for the new interface and compare what it shows vs
> `CONFIG_DB`:
> ```bash
> sonic-db-cli STATE_DB hgetall 'INTERFACE_TABLE|Loopback1|10.10.10.10/32'
> ```
> What is the difference in purpose between `CONFIG_DB` (what we want) and `STATE_DB`
> (what actually exists)?

> **Challenge 4:** After the prefix appears on Spine4, check whether Leaf2 and Leaf3
> also learn `10.10.10.10/32`. If they do — trace the full ASIC_DB next-hop chain on
> Leaf2 the same way we did in the previous section. Does the egress port resolve to
> `Ethernet0` as expected?
