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
watch -n 1 "sonic-db-cli APPL_DB hgetall 'ROUTE_TABLE:10.10.10.10/32'"
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



!NOTE : WORK IN PROGRESS.
