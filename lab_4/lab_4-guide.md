# Cisco 8000 SONiC — ACL & CoPP Lab Guide

This guide combines two hands-on tracks for **SONiC on Cisco 8000**: **Exercise 1** programs and verifies an **L3 ingress ACL** on a front-panel port; **Exercise 2** walks a **read-only CoPP (Control Plane Policing) verification** flow (seed policy, SWSS, APPL_DB, orchestration logs, STATE_DB). Use it for training, acceptance testing, or operational runbooks.

**Format:** Step-by-step structure with objectives, prerequisites, commands, example output, and checklists. (A reference template at `https://github.com/cisco-asp-web/SONiC/blob/main/lab_2/lab_2-guide.md` was not reachable during authoring; this document follows a common lab-guide pattern: overview, learning objectives, environment, discrete exercises, verification.)

---

## Table of contents

- [Learning objectives](#learning-objectives)
- [Lab environment and shared prerequisites](#lab-environment-and-shared-prerequisites)
- [Exercise 1 — L3 ingress ACL: configuration and verification](#exercise-1--l3-ingress-acl-configuration-and-verification)
- [Exercise 2 — CoPP: verification](#exercise-2--copp-verification)

---

## Learning objectives

After completing this lab, you should be able to:

- **Exercise 1 (ACL):** Author SONiC **Config DB** JSON for an **L3** **INGRESS** ACL bound to a port; load it with `config load`; validate with `show acl`, `counterpoll`, `aclshow`, and **Cisco 8000** `show platform npu acl` (with `sudo`).
- **Exercise 2 (CoPP):** Explain how **`copp_cfg.json`** maps **COPP_TRAP** to **COPP_GROUP**; verify **swss** / **orchagent** / **coppmgrd**; inspect **APPL_DB** `COPP_TABLE`, **swss.rec**, and **STATE_DB** `COPP_GROUP_TABLE` for consistency with the seed policy.

---

## Lab environment and shared prerequisites

| Item | Detail |
|------|--------|
| **Platform** | SONiC on **Cisco 8000** |
| **Access** | Operator account with **sudo**; for CoPP, `docker exec` into the **swss** container where noted |
| **Read-only vs change** | **Exercise 1** writes **CONFIG_DB** (config change). **Exercise 2** Verification Steps **1–5** are **read-only**; **Step 6** is policy-change guidance (maintenance window) |
| **Reference outputs** | Example CLI and Redis excerpts illustrate **expected shape**; hostnames, timestamps, PIDs, Redis DB indices, and **orchagent** MAC arguments will differ on your devices |

---

## Exercise 1 — L3 ingress ACL: configuration and verification

### Objectives

- Configure an **L3** **INGRESS** ACL on **Ethernet0** that **drops** IPv4 traffic matching **SRC_IP** `1.1.1.1/32` and **DST_IP** `3.3.3.3/32`.
- Verify the ACL in **`show acl`**, **`aclshow`** / **`counterpoll`**, and **Cisco 8000** **`sudo show platform npu acl`**.

This lab walks through configuring an ingress **L3** ACL on **Ethernet0** and verifying it in the SONiC CLI, ACL counters, and the Cisco 8000 NPU. Commands and **example output** are from a SONiC session on **pod8-leaf3**; counter verification uses test traffic from **Leaf1**.

**ACL JSON:** **Step 2 — ACL JSON on the switch** (below) includes the file contents and a **field-by-field description** of `ACL_TABLE` and `ACL_RULE` (what each key does and how traffic matches).

Each numbered step opens with **why** you run it; bullets before code blocks describe **what each command does** in this lab.

---

### Prerequisites

You need a lab path where traffic can hit the ACL leaf the way the JSON assumes, and the same CLIs as in the capture. This list is the minimum software, access, and topology assumptions before you run the steps.

- SONiC on **Cisco 8000** with `config`, `show acl`, `aclshow`, `counterpoll`, and `sudo show platform npu acl …`
- Shell user **admin** with **sudo**
- Port **Ethernet0** used as the ACL binding on the leaf where the ACL is applied (**pod8-leaf3** in the original capture)
- **Leaf1** can source traffic as **1.1.1.1** toward **3.3.3.3** across the lab fabric (adjust host/interface names to your topology)

---

### Topology (reference)

This section ties names (**Leaf1**, ACL leaf, **Ethernet0**) to the match fields in the JSON so later steps are unambiguous.

Ingress on **Ethernet0** (ACL leaf): drop IPv4 when **SRC_IP** is `1.1.1.1/32` and **DST_IP** is `3.3.3.3/32`.

**Traffic for Step 6:** initiated from **Leaf1** so replies or echo requests use the lab addresses the ACL matches:

```bash
ping 3.3.3.3 -I 1.1.1.1
```

- **`ping`** — Generates ICMP echo traffic toward `3.3.3.3`.
- **`-I 1.1.1.1`** — On Linux, binds the source address (and often the egress interface) so packets leave Leaf1 as **1.1.1.1** toward **3.3.3.3**; use the equivalent on your OS if needed.

---

### Step 1 — Baseline (optional)

Confirm the switch has **no** ACL tables/rules (or note what is already present) so you can tell the lab config apart from any pre-existing state. Skip if you already know the device is clean.

- **`show acl rule`** — Lists every ACL rule from the running/orchestrated view: table, rule name, priority, action, match fields, and status (e.g. Active).

```bash
show acl rule
```

**Output (before load):**

```text
Table    Rule    Priority    Action    Match    Status
-------  ------  ----------  --------  -------  --------
```

- **`show acl table`** — Lists ACL tables: type (L3/L2), which ports they bind to, stage (ingress/egress), description, and status.

```bash
show acl table
```

**Output (before load):**

```text
Name    Type    Binding    Description    Stage    Status
------  ------  ---------  -------------  -------  --------
```

---

### Step 2 — ACL JSON on the switch

This step has two parts: the JSON you load into Config DB, then a **description of every field** (tables and behavior summary).

- **`cat /tmp/acl.json`** — Prints the file so you can proofread keys and values before loading (path is arbitrary; `/tmp` is common on switches).

```bash
cat /tmp/acl.json
```

**Output:**

```json
{
    "ACL_TABLE": {
        "ACL_DENY": {
            "policy_desc": "ACL DENY",
            "ports": [
                "Ethernet0"
            ],
            "stage": "INGRESS",
            "type": "L3"
        }
    },
    "ACL_RULE": {
        "ACL_DENY|10-DENY": {
            "PRIORITY": "10",
            "IP_TYPE": "IP",
            "SRC_IP": "1.1.1.1/32",
            "DST_IP": "3.3.3.3/32",
            "PACKET_ACTION": "DROP"
        }
    }
}
```

*(Create this file with `cat > /tmp/acl.json <<'EOF'` … `EOF` if you are building it from scratch; that here-document writes the JSON to disk without an external editor.)*

### Description of the ACL JSON (field reference)

This JSON is a **SONiC Config DB** fragment that defines **one L3 ingress ACL** bound to a front-panel port and **one IPv4 drop rule**.

#### `ACL_TABLE` → `ACL_DENY`

| Field | Meaning |
|--------|--------|
| **`ACL_TABLE`** | Top-level object for ACL *tables* (where ACLs attach and which pipeline stage they use). |
| **`ACL_DENY`** | Name of this ACL table. All rules for it use keys like `ACL_DENY\|<rule-name>`. |
| **`policy_desc`** | Human-readable string (`ACL DENY`); for operators / `show acl table`, not used for matching. |
| **`ports`** | Interfaces the table is **bound** to. Here **`Ethernet0`**: the ACL is evaluated on **ingress** traffic arriving on that port. |
| **`stage`** | **`INGRESS`**: rules run on traffic **into** the listed ports (before normal forwarding for that packet on that hop). |
| **`type`** | **`L3`**: IPv4/IPv6-style fields (e.g. SIP/DIP) are allowed; not a pure L2 MAC/VLAN ACL table. |

**Summary:** On **`Ethernet0`**, at **ingress**, apply L3 table **`ACL_DENY`**.

#### `ACL_RULE` → `ACL_DENY|10-DENY`

| Field | Meaning |
|--------|--------|
| **`ACL_RULE`** | Top-level object for individual **rules**; each key ties a rule to a table. |
| **`ACL_DENY|10-DENY`** | Rule **`10-DENY`** in table **`ACL_DENY`** (SONiC’s `TABLE|RULE` naming). |
| **`PRIORITY`** | **`10`**: rule ordering vs other rules in the same table (lower number = higher precedence in typical SONiC ACL displays). |
| **`IP_TYPE`** | **`IP`**: match **IPv4** (as opposed to IPv6-only or other classifications, depending on image). |
| **`SRC_IP`** | **`1.1.1.1/32`**: source address must be exactly **1.1.1.1**. |
| **`DST_IP`** | **`3.3.3.3/32`**: destination address must be exactly **3.3.3.3**. |
| **`PACKET_ACTION`** | **`DROP`**: matching packets are **silently discarded** at this ACL application point (no forward out that path for this rule’s hit). |

**Behavior summary:** Any **IPv4** packet entering **`Ethernet0`** whose **source is 1.1.1.1** and **destination is 3.3.3.3** matches **`10-DENY`** and is **dropped** at **ingress** on that port, under table **`ACL_DENY`**.

---

### Step 3 — Load into Config DB

Merge the JSON into **Config DB** so orchagent and related daemons can program the ASIC. **`sudo`** is required to write system configuration; **`-y`** skips interactive confirmation (omit it if you prefer prompts).

- **`sudo config load <file> -y`** — Runs `sonic-cfggen` to apply JSON keys to Redis Config DB; only keys present in the file are merged (behavior may vary by image for deletes).

```bash
sudo config load /tmp/acl.json -y
```

**Output:**

```text
Running command: /usr/local/bin/sonic-cfggen -j /tmp/acl.json --write-to-db
```

---

### Step 4 — Verify logical ACL

After load, confirm SONiC’s **control-plane view** of ACLs matches intent: table bound to the right port, rule **Active**, action **DROP**, and match fields correct. These `show acl` commands read state the same way operators troubleshoot (no ASIC debug socket).

- **`show acl rule`** — All rules, or use optional table name / `--verbose` for one table (see below).

```bash
show acl rule
```

**Output:**

```text
Table     Rule     Priority    Action    Match               Status
--------  -------  ----------  --------  ------------------  --------
ACL_DENY  10-DENY  10          DROP      DST_IP: 3.3.3.3/32  Active
                                         IP_TYPE: IP
                                         SRC_IP: 1.1.1.1/32
```

- **`show acl table`** — Summarizes each ACL table’s bindings and stage after programming.

```bash
show acl table
```

**Output:**

```text
Name      Type    Binding    Description      Stage    Status
--------  ------  ---------  ---------------  -------  --------
ACL_DENY  L3      Ethernet0  ACL DENY           ingress  Active
```

- **`show acl rule ACL_DENY --verbose`** — Narrows output to table **ACL_DENY** and invokes `acl-loader show rule` under the hood for the same logical view with optional extra detail depending on image.

```bash
show acl rule ACL_DENY --verbose
```

**Output:**

```text
Running command: acl-loader show rule ACL_DENY
Table     Rule     Priority    Action    Match               Status
--------  -------  ----------  --------  ------------------  --------
ACL_DENY  10-DENY  10          DROP      DST_IP: 3.3.3.3/32  Active
                                         IP_TYPE: IP
                                         SRC_IP: 1.1.1.1/32
```

---

### Step 5 — Counter polling

ACL **hit counters** in SONiC are updated by a polling process. This step confirms **ACL** counter polling is **enabled** and how often it runs; without that, `aclshow` may look stale or empty depending on timing.

- **`counterpoll show`** — Displays all counter poll types, interval in milliseconds, and enable/disable status.

```bash
counterpoll show
```

**Output:**

```text
Type                        Interval (in ms)    Status
--------------------------  ------------------  --------
QUEUE_STAT                  default (10000)     enable
PORT_STAT                   default (1000)      enable
PORT_BUFFER_DROP            default (60000)     enable
RIF_STAT                    default (1000)      enable
QUEUE_WATERMARK_STAT        default (60000)     enable
PG_WATERMARK_STAT           default (60000)     enable
PG_DROP_STAT                default (10000)     enable
BUFFER_POOL_WATERMARK_STAT  default (60000)     enable
ACL                         10000               enable
```

---

### Step 6 — Verify ACL counters (`aclshow`)

**`aclshow`** reads **ACL counter** statistics (packets/bytes per rule) that the dataplane accumulates for matched traffic. The flow below: inspect counters, clear them, generate lab traffic from **Leaf1**, then confirm counts rise on the ACL leaf.

- **`aclshow -t ACL_DENY`** — Shows counters only for rules in table **ACL_DENY** (useful when many ACL tables exist). Run it first to baseline hits on this table.

```bash
aclshow -t ACL_DENY
```

**Example output (while hits exist):**

```text
RULE NAME    TABLE NAME      PRIO    PACKETS COUNT    BYTES COUNT
-----------  ------------  ------  ---------------  -------------
10-DENY      ACL_DENY          10               62           6324
```

- **`aclshow`** — Lists all ACL rules that have counter entries (in a small lab, often the same rows as `-t` for a single table).

Verify all ACL counters (same data when only one rule is active):

```bash
aclshow
```

**Example output:**

```text
RULE NAME    TABLE NAME      PRIO    PACKETS COUNT    BYTES COUNT
-----------  ------------  ------  ---------------  -------------
10-DENY      ACL_DENY          10              229          23358
```

- **`aclshow -c`** — Clears ACL counter statistics in SONiC so the next measurement starts from zero (run on the **ACL leaf** where you read counters).

Clear ACL statistics so the next test starts from zero:

```bash
aclshow -c
```

**Example output right after clear (no hits yet):**

```bash
aclshow
```

```text
RULE NAME    TABLE NAME    PRIO    PACKETS COUNT    BYTES COUNT
-----------  ------------  ------  ---------------  -------------
```

From **Leaf1**, generate traffic that matches the rule (source **1.1.1.1**, destination **3.3.3.3**). Same command as in [Topology](#topology-reference); ICMP is carried in IPv4, so echo traffic increments the drop counter if those IPv4 headers match the ACL at the bind point.

```bash
ping 3.3.3.3 -I 1.1.1.1
```

On the ACL **leaf** (where **Ethernet0** sees this flow), confirm counters increase again:

```bash
aclshow
```

**Example output after traffic:**

```text
RULE NAME    TABLE NAME      PRIO    PACKETS COUNT    BYTES COUNT
-----------  ------------  ------  ---------------  -------------
10-DENY      ACL_DENY          10               13           1326
```

- **`aclshow -t ACL_DENY`** (again) — Re-check the same table after traffic; packet/byte totals should climb if the path hits this rule.

```bash
aclshow -t ACL_DENY
```

**Example output:**

```text
RULE NAME    TABLE NAME      PRIO    PACKETS COUNT    BYTES COUNT
-----------  ------------  ------  ---------------  -------------
10-DENY      ACL_DENY          10               91           9282
```

> **Note:** **`-r`** filters by **rule name** (e.g. `10-DENY`), not by table. Use **`aclshow -r 10-DENY`** if you want rule-specific counters.

- **`aclshow -h`** — Prints option summary (`-a` all, `-c` clear, `-r` rules, `-t` tables, verbosity).

```bash
aclshow -h
```

**Output:**

```text
usage: aclshow [-h] [-a] [-c] [-r RULES] [-t TABLES] [-v] [-vv]

Display SONiC switch Acl Rules and Counters

options:
  -h, --help            show this message and exit
  -a, --all             Show all ACL counters
  -c, --clear           Clear ACL counters statistics
  -r RULES, --rules RULES
                        action by specific rules list: Rule1_Name,Rule2_Name
  -t TABLES, --tables TABLES
                        action by specific tables list: Table1_Name,Table2_Name
  -v, --version         show program's version number and exit
  -vv, --verbose        Verbose output
```

---

### Step 7 — Verify in hardware (NPU)

Cisco 8000 exposes **ASIC-level** ACL bind points and ACE contents via **`show platform npu acl`**. That path talks to a **debug/SDK context** and normally requires **root/sudo**. Use it to prove the hardware programmed SIP/DIP and to capture the dynamic **ACL ID** for deep dives.

Without **sudo** (expect failure on images that require privileged ASIC access):

```bash
show platform npu acl summary
```

**Output:**

```text
cannot connect to debug shell socket asic 0. Are you using sudo?
```

- **`sudo show platform npu acl summary`** — High-level view: L3 port ACL bind (which ingress IPv4 ACL instance is on the port), TCAM/resource summary, and **ACE table** rows (SIP/DIP masks as programmed).

With **sudo**:

```bash
sudo show platform npu acl summary
```

**Output:**

```text
show acl all
ACL is not binding to L2 SERVICE PORT
 +---------------------------------------------+
 |               L3 PORT ACL Bind              |
 +---------------------+-----------------------+
 | L3 AC PORT(gid:oid) | Ingress IPv4 ACL ID/0 |
 +---------------------+-----------------------+
 |      0x404:618      |         10483         |
 +---------------------+-----------------------+
 +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 |                                                                                                                                                   ACL Table                                                                                                                                                    |
 +--------+------------+----------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------+
 | ACL ID | ACE Number | Key Type |      Available       |                                                                                                Key Profile                                                                                                 |              Command Profile              |
 +--------+------------+----------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------+
 | 10481  |     0      | ETHERNET | 18446744073709551615 |                               Direction = {INGRESS},TCAM interface = {E_0},Resoure Type = {INGRESS_IPV6_NARROW_DB2_INTERFACE0_ACL},Fields = {ETHER_TYPE SOURCE_SYSTEM_PORT }                               | Actions = {                             } |
 | 10483  |     1      |   IPV4   |        32767         | Direction = {INGRESS},TCAM interface = {E_0},Resoure Type = {INGRESS_IPV6_NARROW_DB2_INTERFACE1_ACL},Fields = {TOS PROTOCOL IPV4_SIP IPV4_DIP SPORT DPORT MSG_CODE MSG_TYPE TCP_FLAGS SOURCE_SYSTEM_PORT } | Actions = {                             } |
 +--------+------------+----------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------+
 +------------------------------------------------------------------------------+
 |                                  ACE Table                                   |
 +--------+----------+-------------------------+-------------------------+------+
 | ACL ID | Position |       IPV4_SIP(F)       |       IPV4_DIP(F)       | (C)  |
 +--------+----------+-------------------------+-------------------------+------+
 | 10483  |    0     | 1.1.1.1/255.255.255.255 | 3.3.3.3/255.255.255.255 | True |
 +--------+----------+-------------------------+-------------------------+------+
```

- **`sudo show platform npu acl ace -a <acl_id> -p <position>`** — Dumps the ACE at **position** inside **acl_id** (IDs come from the summary output; **10483** / **0** match this lab capture only).

```bash
sudo show platform npu acl ace -a 10483 -p 0
```

**Output:**

```text
show acl ace 10483 0 detail
 +------------------------------------------------------------------------------+
 |                                   ACL ACE                                    |
 +--------+----------+-------------------------+-------------------------+------+
 | ACL ID | Position |       IPV4_SIP(F)       |       IPV4_DIP(F)       | (C)  |
 +--------+----------+-------------------------+-------------------------+------+
 | 10483  |    0     | 1.1.1.1/255.255.255.255 | 3.3.3.3/255.255.255.255 | True |
 +--------+----------+-------------------------+-------------------------+------+
```

> **Note:** **ACL ID** (here **10483**) comes from your device’s summary; it may differ on another switch or after reprogramming.

---

### Step 8 — Rollback / cleanup (optional)

When the lab is done, remove the ACL so it does not affect later work. SONiC images differ: some support **`config acl`** (or similar) to remove tables/rules; others use **`config`** to delete Config DB keys, or a curated JSON that removes **`ACL_TABLE`** / **`ACL_RULE`** entries. Prefer the documented method for your build so you do not leave partial state in Redis.

---

### Verification checklist

Use this list as a quick pass/fail gate; each line maps to a step above.

- [ ] `show acl table` shows **ACL_DENY** on **Ethernet0**, **Active**
- [ ] `show acl rule` shows **10-DENY**, **DROP**, matches and **Active**
- [ ] `counterpoll show` shows **ACL** `enable`
- [ ] `aclshow -t ACL_DENY` and `aclshow` show expected counts; `aclshow -c` resets; **Leaf1** `ping 3.3.3.3 -I 1.1.1.1` drives new hits on the ACL leaf
- [ ] `sudo show platform npu acl summary` / `sudo show platform npu acl ace -a <acl_id> -p 0` match SIP/DIP in hardware

---

## Exercise 2 — CoPP: verification

### Objectives

- Confirm **seed CoPP policy** on disk and trace it through **SWSS**, **APPL_DB** `COPP_TABLE`, **`swss.rec`**, and **STATE_DB** `COPP_GROUP_TABLE`.
- Understand when **read-only** verification ends and **policy changes** require a controlled change window.

This document is a **repeatable, read-only verification procedure** for **Control Plane Policing (CoPP)** on **Cisco 8000** SONiC. Use it after **installation**, **software upgrade**, **CONFIG_DB changes affecting CoPP**, or when **investigating control-plane instability** (BGP churn, neighbor loss, CPU punt pressure). It confirms that **seed policy**, **SWSS daemons**, **APPL_DB programming**, **orchestration logs**, and **STATE_DB health** are consistent.

**Theory:** The section **CoPP theory — configuration model and default behavior** explains **`copp_cfg.json`**, the **`default`** catch-all, **trap vs copy**, **SR_TCM** metering, and a **protocol-by-protocol** reading of Q200-style defaults. Read it once for context; the verification steps below do not require changing configuration.

**Reference output:** Example CLI and Redis excerpts illustrate **expected shape** on a validated Cisco 8000 image. Your hostname, timestamps, PIDs, and MAC arguments to **orchagent** will differ. **Always compare** live output to **`/etc/sonic/copp_cfg.json`** on the same device.

**Structure:** Each numbered step states **why** it belongs in this verification flow; bullets before code blocks state **what the command proves**.

---

### Prerequisites

- **Platform:** SONiC on **Cisco 8000** with **`show feature`**, **`redis-cli`** (and preferably **`sonic-db-cli`** per your runbook).
- **Access:** Operator account with **`docker exec`** into the **swss** container (read-only checks inside SWSS).
- **Change window:** **Verification Steps 1–5** are **non-disruptive**. **Verification Step 6** (policy changes) requires a **controlled maintenance window**, **rollback plan**, and **service-owner approval**—incorrect CoPP can drop **BGP**, **LACP**, or **ARP** to the CPU path.

---

### What CoPP does (one paragraph)

CoPP maps **ASIC trap classes** (BGP, LACP, ARP, LLDP, packets to local IPs, and so on) to **policer parameters** (CIR/CBS), **CPU queue**, **trap vs copy**, and **priority**. The goal is to **protect the control plane** from floods while keeping protocols stable. This is **not** the same as a front-panel **L3 ACL** on data-plane traffic.

---

### CoPP theory — configuration model and default behavior

This section explains **how SONiC expresses CoPP policy in JSON**, how **groups** relate to **traps**, what the **`default`** group is for, and how to read the **Cisco SONiC / Q200-style** defaults shipped under **`/etc/sonic/copp_cfg.json`**. Use it as **background** before executing the verification procedure.

#### Policy file and the two top-level objects

On SONiC, baseline CoPP is commonly seeded from **`/etc/sonic/copp_cfg.json`**. That file has two cooperating parts:

1. **`COPP_GROUP`** — Defines **named buckets** of behavior: how packets are **trapped or copied**, which **CPU queue** they use, and how they are **metered** (policer type, mode, CIR/CBS, exceed action). Think of a group as **“how the ASIC should treat this class of CPU-bound packets.”**
2. **`COPP_TRAP`** — Maps **protocol-oriented trap entries** to a **`trap_group`** (the name of a **`COPP_GROUP`**). Each trap entry lists one or more **`trap_ids`** (ASIC-level trap identifiers such as **`bgp`**, **`lacp`**, **`arp_req`**). Those IDs are the **control-plane packet types** the policer and queue logic attach to.

**End-to-end mental model:** *Trap IDs* (what the packet **is**) → *COPP_TRAP row* (which **group** it joins) → *COPP_GROUP* ( **trap vs copy**, **queue**, **CIR/CBS**, **drop on red**).

#### The `default` group (catch-all for “other” CPU traffic)

The group named **`default`** defines how **control-plane traffic that does not match any `trap_ids` wired through `COPP_TRAP`** is handled. In other words, if a packet is **destined for the CPU** (punted/trapped by the ASIC) but **does not** correspond to an explicit trap mapping such as BGP, LACP, ARP, LLDP, and so on, it falls under **`default`**.

On many images the **`default`** object specifies metering (**`cir`**, **`cbs`**), **`meter_type`**, **`mode`**, **`queue`**, and **`red_action`**. Some builds also set **`trap_action`** explicitly on **`default`**; others rely on implicit trap behavior. Always **`cat`** the file on **your** image rather than assuming optional keys.

Typical **`default`** intent: a **moderate** policer on **queue 0** so miscellaneous punt traffic cannot starve well-known protocols that have their own, tighter or looser, groups.

#### Default `COPP_TRAP` names on Cisco SONiC (reference list)

The seed JSON discussed in this guide defines named traps such as **`bgp`**, **`lacp`**, **`arp`**, **`lldp`**, **`dhcp_relay`**, **`udld`**, **`ip2me`**, **`macsec`**, **`nat`**, and **`sflow`**. These are **policy containers**: the real ASIC programming uses the **`trap_ids`** strings inside each container. Your platform may add or omit traps by feature; treat this list as the **Q200 reference baseline**, not a guarantee on every SKU or branch.

A single **Q200-style** example of the full **`COPP_GROUP`** / **`COPP_TRAP`** JSON appears under **Verification Step 1 — Confirm seed policy on disk** (the **Reference output** for `cat /etc/sonic/copp_cfg.json`). Use that block as the structural baseline; your on-switch file may differ slightly (for example the **`default`** object may include **`trap_action`**).

#### Protocol-by-protocol reading of the default map

Each **`COPP_TRAP`** row points at one **`trap_group`**. **Multiple trap names can share one group**, which means they **share one policer instance** after coppmgr merges them (see **`swss.rec`** / **`COPP_TABLE`** in **Verification Steps 3 and 4**). The table below summarizes the **intent** of the Q200 defaults.

| Named trap | `trap_ids` (ASIC classes) | `trap_group` | CPU queue (from group) | Policer (packets) | Notes |
|------------|---------------------------|--------------|-------------------------|-------------------|--------|
| **bgp** | `bgp`, `bgpv6` | `queue4_group1` | 4 | CIR **6000**, CBS **6000** | **`trap_action: trap`** — punt to CPU for the routing stack; excess **dropped** by **`red_action: drop`**. |
| **lacp** | `lacp` | `queue4_group1` | 4 | same as BGP | Shares **queue4_group1** meter with BGP/MACsec traps that land in the same merged group. **`always_enabled: true`**. |
| **arp** | `arp_req`, `arp_resp`, `neigh_discovery` | `queue4_group2` | 4 | CIR **600**, CBS **600** | **`trap_action: copy`** — a **copy** is delivered to the CPU path for neighbor learning while the **original** can still follow normal **bridging/L3 forwarding** semantics where the ASIC supports it. **`always_enabled: true`**. |
| **lldp** | `lldp` | `queue4_group3` | 4 | CIR **100**, CBS **100** | Tight policer; often merged with UDLD/DHCP in APPL_DB as one row. |
| **dhcp_relay** | `dhcp`, `dhcpv6` | `queue4_group3` | 4 | same as LLDP | Same group as LLDP/UDLD → **shared** 100 pps budget in the seed. |
| **udld** | `udld` | `queue4_group3` | 4 | same | **`always_enabled: true`**. |
| **ip2me** | `ip2me` | `queue1_group1` | 1 | CIR **6000**, CBS **6000** | **Packets destined to the switch’s own IP addresses** (my-ip exception path). High budget relative to discovery protocols. **`always_enabled: true`**. |
| **macsec** | `eapol` | `queue4_group1` | 4 | same as BGP | 802.1X **EAPoL** to CPU; shares **queue4_group1** with BGP/LACP. |
| **nat** | `src_nat_miss`, `dest_nat_miss` | `queue1_group2` | 1 | CIR **600**, CBS **600** | NAT **miss** traps toward control plane; separate from **ip2me**’s higher tier. |
| **sflow** | `sample_packet` | `queue2_group1` | 2 | CIR **1000**, CBS **1000** | Sampling path; includes **`genetlink_name` / `genetlink_mcgrp_name`** for **psample**-style delivery toward the kernel/user sampling pipeline. |

**Takeaway:** **Queue 4** carries much of **L2/L3 control adjacency** (BGP, LACP, MACsec, ARP copy). **Queue 1** splits **“my IP”** (**ip2me**, high) from **NAT miss** (lower). **Queue 2** is reserved for **sFlow**-like sampling in this profile.

#### `COPP_GROUP` parameters (theory)

| Parameter | Meaning |
|-----------|--------|
| **`trap_action`** | **`trap`** — Intercept the packet and send it on the **CPU / punt** path; it does **not** continue as normal forwarded data-plane traffic in the pure-trap case. Use for protocols the **switch must terminate** (for example BGP updates to **local** BGP). **`copy`** — The ASIC **duplicates** the packet: one copy to the CPU for **protocol or learning** logic, while the **original** can still be **forwarded** where hardware and SAI semantics allow. This pattern is **common for ARP/ND** so the CPU can learn neighbors without necessarily swallowing the only copy. Exact behavior is **SAI/ASIC-specific**; validate on your release if you rely on copy semantics. |
| **`trap_priority`** | Numeric **priority among traps** at the ASIC. Higher values usually mean **more urgent** handling when multiple traps compete; treat the number as **vendor-relative**, not portable across unrelated chips. |
| **`queue`** | **CPU queue index** after trap/copy. Linux/HSQM scheduling can give **different scheduling weights** per queue so **BGP/LACP** can be isolated from **LLDP** floods. |
| **`meter_type`** | **`packets`** — **`cir`** and **`cbs`** count **packets** (not bytes). Some designs support byte meters; this seed uses **packet** policing. |
| **`mode`** | **`sr_tcm`** — **Single Rate Three Color Marker** style policing. Conceptually the meter marks packets **green** (within committed rate), **yellow** (burst use depending on implementation), or **red** (exceeds allowed burst / committed budget). **SONiC + SAI** map these outcomes to **`red_action`**. |
| **`cir`** | **Committed Information Rate** — sustained allowance (here, **packets per second**). |
| **`cbs`** | **Committed Burst Size** — burst allowance in **packets** (paired with **`cir`** for token-bucket behavior). |
| **`red_action`** | **`drop`** — Packets classified as **“red”** (over limit) are **dropped** instead of being sent to the CPU. |
| **`genetlink_name`** / **`genetlink_mcgrp_name`** | Used on **`queue2_group1`** for **sFlow** / **psample** integration so sampled packets can reach the correct **generic netlink** channel in Linux. |

#### `COPP_TRAP` parameters (theory)

| Parameter | Meaning |
|-----------|--------|
| **`trap_ids`** | Comma-separated **ASIC trap identifiers** installed under this named trap. |
| **`trap_group`** | Name of a **`COPP_GROUP`** entry: inherits **`trap_action`**, **`queue`**, **`cir`**, **`cbs`**, and related fields. |
| **`always_enabled`** | When **`true`**, SONiC should keep this trap **active** regardless of whether the high-level feature looks “configured” from the operator’s perspective. Important for **fundamental** protocols (**LACP**, **ARP/ND**, **UDLD**, **ip2me**) so the ASIC still punts required packet types during bring-up. This key lives on **`COPP_TRAP`** rows in JSON (not inside **`COPP_GROUP`**). |

#### Design notes operators care about

- **Shared policers:** Traps that reference the **same** `trap_group` name share **one** policer after merge (for example **LLDP + DHCP relay + UDLD** all on **`queue4_group3`** → one **100 pps** budget in the seed). Raising CIR for “LLDP only” without splitting groups requires a **config model** change, not a one-line tweak inside the same group.
- **Why ARP uses `copy`:** Neighbor discovery benefits from **CPU visibility** without necessarily blocking **normal forwarding** of the same frame where **copy** semantics apply.
- **Why BGP and LACP share a group:** Both are **wired-speed-sensitive** control protocols on many designs; the **6000 pps** tier reflects a **high-trust** class. Under heavy BGP churn, validate offered **pps** to the CPU against **CIR/CBS** and CoPP drop counters (platform-specific) before changing policy.

---

### Architecture (reference)

| Stage | Artifact | Role |
|--------|-----------|------|
| Seed | **`/etc/sonic/copp_cfg.json`** | Default **`COPP_GROUP`** + **`COPP_TRAP`** template shipped with the image. |
| Persistent config | **`/etc/sonic/config_db.json`** (if present) | May override or extend CoPP; empty/missing CoPP keys often mean the seed applies. |
| Effective policy | **CONFIG_DB** (Redis) | What **coppmgrd** reconciles after merges and feature state. |
| Orch input | **APPL_DB** `COPP_TABLE:<group>` | What **orchagent** programs via **SAI**. On many images **`redis-cli -n 0`** is **APPL_DB**—**confirm** with **`sonic-db-cli`** / platform documentation before scripting. |
| Runtime status | **STATE_DB** `COPP_GROUP_TABLE|…` | Operational state such as **`state: ok`**. **STATE_DB** Redis index varies by release (**`-n 6`** in the reference capture)—**verify** on target software. |

Daemons:

- **`coppmgrd`** — CoPP manager: CONFIG_DB → policy toward orchestration.
- **`orchagent`** — Contains **COPP orchestration** path that programs the ASIC through **syncd** / vendor **SAI**.

---

### Verification Step 1 — Confirm seed policy on disk

**Why:** Verification starts from the **authoritative seed** merged into CONFIG_DB. You record **CIR/CBS**, **queues**, **trap_action**, and **which trap_ids share a group** so later Redis checks have a baseline.

- **`cat /etc/sonic/copp_cfg.json`** — Displays the image **CoPP baseline** (compare every field in Steps 3–4 to this file).

```bash
cat /etc/sonic/copp_cfg.json
```

**Reference output (baseline file; must match your image version):**

```json
{
    "COPP_GROUP": {
            "default": {
                    "trap_action":"trap",
                    "queue": "0",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"600",
                    "cbs":"600",
                    "red_action":"drop"
            },
            "queue4_group1": {
                    "trap_action":"trap",
                    "trap_priority":"4",
                    "queue": "4",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"6000",
                    "cbs":"6000",
                    "red_action":"drop"
            },
            "queue4_group2": {
                    "trap_action":"copy",
                    "trap_priority":"4",
                    "queue": "4",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"600",
                    "cbs":"600",
                    "red_action":"drop"
            },
            "queue4_group3": {
                    "trap_action":"trap",
                    "trap_priority":"4",
                    "queue": "4",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"100",
                    "cbs":"100",
                    "red_action":"drop"
            },
            "queue1_group1": {
                    "trap_action":"trap",
                    "trap_priority":"1",
                    "queue": "1",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"6000",
                    "cbs":"6000",
                    "red_action":"drop"
            },
            "queue1_group2": {
                    "trap_action":"trap",
                    "trap_priority":"1",
                    "queue": "1",
                    "meter_type":"packets",
                    "mode":"sr_tcm",
                    "cir":"600",
                    "cbs":"600",
                    "red_action":"drop"
            },
            "queue2_group1": {
                    "cbs": "1000",
                    "cir": "1000",
                    "genetlink_mcgrp_name": "packets",
                    "genetlink_name": "psample",
                    "meter_type": "packets",
                    "mode": "sr_tcm",
                    "queue": "2",
                    "red_action": "drop",
                    "trap_action": "trap",
                    "trap_priority": "1"
            }
    },
    "COPP_TRAP": {
            "bgp": {
                    "trap_ids": "bgp,bgpv6",
                    "trap_group": "queue4_group1"
            },
            "lacp": {
                    "trap_ids": "lacp",
                    "trap_group": "queue4_group1",
                    "always_enabled": "true"
            },
            "arp": {
                    "trap_ids": "arp_req,arp_resp,neigh_discovery",
                    "trap_group": "queue4_group2",
                    "always_enabled": "true"
            },
            "lldp": {
                    "trap_ids": "lldp",
                    "trap_group": "queue4_group3"
            },
            "dhcp_relay": {
                    "trap_ids": "dhcp,dhcpv6",
                    "trap_group": "queue4_group3"
            },
            "udld": {
                    "trap_ids": "udld",
                    "trap_group": "queue4_group3",
                    "always_enabled": "true"
            },
            "ip2me": {
                    "trap_ids": "ip2me",
                    "trap_group": "queue1_group1",
                    "always_enabled": "true"
            },
            "macsec": {
                    "trap_ids": "eapol",
                    "trap_group": "queue4_group1"
            },
            "nat": {
                    "trap_ids": "src_nat_miss,dest_nat_miss",
                    "trap_group": "queue1_group2"
            },
            "sflow": {
                    "trap_group": "queue2_group1",
                    "trap_ids": "sample_packet"
            }
    }
}
```

#### Field reference — `COPP_GROUP`

Each **group** is a **policer + queue + trap behavior** bundle. Multiple **traps** can reference the same group (they then share the meter).

| Field | Meaning |
|--------|--------|
| **`trap_action`** | **`trap`** — send matching control packets to the CPU path. **`copy`** — often used where the ASIC should **copy** samples (e.g. ARP/ND) per vendor/SAI semantics; do not assume identical to **`trap`** on every ASIC without checking behavior in your release notes. |
| **`trap_priority`** | Relative ordering among traps (string numeric in JSON). |
| **`queue`** | CPU **queue index** for this class after trapping. |
| **`meter_type`** | **`packets`** here — policer counts **packets** (not bytes). |
| **`mode`** | **`sr_tcm`** — single-rate three-color marker style token bucket (committed/excess semantics; **`red_action`** defines exceed behavior). |
| **`cir` / `cbs`** | **Committed information rate** (packets/sec in this file) and **committed burst** (packets). Excess is subject to **`red_action`**. |
| **`red_action`** | **`drop`** — drop on exceed (policing). |
| **`genetlink_name` / `genetlink_mcgrp_name`** | Present on **`queue2_group1`** for **sFlow** / **psample** style sampling toward user space; typical for **sample_packet** path. |

#### Field reference — `COPP_TRAP`

Each entry names a **feature-facing trap bucket** and points at a **`trap_group`**.

| Field | Meaning |
|--------|--------|
| **`trap_ids`** | Comma-separated **ASIC trap identifiers** installed under this policy (e.g. **`bgp,bgpv6`**). |
| **`trap_group`** | Name of a **`COPP_GROUP`** entry: inherits CIR/CBS/queue/action. |
| **`always_enabled`** | When **`true`**, trap should stay enabled irrespective of certain feature toggles (image-dependent; treat as **hint** to read coppmgrd / CONFIG_DB behavior for your branch). |

**Operational note — shared policers:** In this JSON, **`lldp`** and **`udld`** both use **`queue4_group3`** (CIR/CBS **100**). **`swss.rec`** (**Verification Step 4**) should show a **single** `COPP_TABLE` row for **`queue4_group3`** with merged **`trap_ids`** (for example **`lldp,udld`** plus other members of that group after merge).

---

### Verification Step 2 — Confirm SWSS feature and processes

**Why:** CoPP is applied by **orchagent** under **swss**. If **swss** is disabled or **orchagent** / **coppmgrd** is not running, **APPL_DB** / **STATE_DB** views are **not trustworthy** for hardware parity.

On the **host** (not inside the container), **`show feature`** confirms **swss** is enabled:

```bash
show feature config | grep -E "Feature|swss"
```

**Output:**

```text
Feature         State            AutoRestart     Owner
swss            enabled          enabled         local
```

```bash
show feature status | grep -E "Feature|swss"
```

**Output:**

```text
Feature         State            AutoRestart     SetOwner
swss            enabled          enabled
```

Inside **SWSS** (orchagent and coppmgrd run here):

- **`docker exec -it swss bash`** — Opens a shell in the **swss** container.
- **`ps -eaf | grep -E 'coppmgrd|orchagent'`** — Shows **coppmgrd** and **orchagent** PIDs and arguments.

```bash
docker exec -it swss bash
ps -eaf | grep -E 'coppmgrd|orchagent'
```

**Output:**

```text
root          59       1  0 06:03 pts/0    00:00:44 /usr/bin/orchagent -d /var/log/swss -t 300-b 1024 -s -m 78:CA:F8:BB:A0:00
root          85       1  0 06:03 pts/0    00:00:03 /usr/bin/coppmgrd
```

- **`supervisorctl status`** — Confirms **supervisord** believes the processes are **RUNNING**.

```bash
supervisorctl status | grep -E 'orchagent|coppmgrd'
```

**Output:**

```text
coppmgrd                         RUNNING   pid 85, uptime 12:52:45
orchagent                        RUNNING   pid 59, uptime 12:52:47
```

**Note:** `show` is a **host** CLI. Inside the **swss** container it may be missing (`bash: show: command not found`); exit with **`exit`** and run **`show …`** on the switch Linux host.

#### Paste safety (optional but important)

If the shell prints **`bash: $'\342\200\213': command not found`** before a valid line, the prompt or paste likely contains a **Unicode zero-width space (U+200B)**. Retype the command in a plain-text terminal or paste from a raw snippet; invisible characters are not valid commands.

---

### Verification Step 3 — APPL_DB `COPP_TABLE` (what orchagent consumed)

**Why:** **`COPP_TABLE`** in **APPL_DB** is the flattened **group** view with **`trap_ids`** attached. This is the primary check that **coppmgrd** merged traps correctly and that **CIR/CBS/queue/trap_action** match the intended policy before it reaches **SAI**.

On the **device under verification**, query **APPL_DB** (below uses **`redis-cli -n 0`** where that maps to **APPL_DB** on your image—**confirm** before automation):

```bash
redis-cli -n 0 hgetall "COPP_TABLE:queue4_group1"
```

**Output:**

```text
 1) "cbs"
 2) "6000"
 3) "cir"
 4) "6000"
 5) "meter_type"
 6) "packets"
 7) "mode"
 8) "sr_tcm"
 9) "queue"
10) "4"
11) "red_action"
12) "drop"
13) "trap_action"
14) "trap"
15) "trap_priority"
16) "4"
17) "trap_ids"
18) "bgp,bgpv6,lacp"
```

Spot-check a single field:

```bash
redis-cli -n 0 hget "COPP_TABLE:queue4_group1" "cir"
```

**Output:**

```text
"6000"
```

> **Note:** Redis **DB index** mapping (**0** for APPL_DB and **6** for STATE_DB in the reference capture) **varies by SONiC branch**. For automated scripts, use **`sonic-db-cli`** with logical database names where supported.

---

### Verification Step 4 — Correlate with `swss.rec` (time-ordered orch trace)

**Why:** **`/var/log/swss/swss.rec`** records **SET** operations on orch-facing tables. It provides a **time-ordered audit trail** of when CoPP rows were pushed and **which field set** orchagent last applied—useful after **reload**, **swss restart**, or **incident** correlation.

```bash
grep -i copp /var/log/swss/swss.rec
```

**Reference output (abbreviated; timestamps and ordering from your device):**

```text
2026-04-20.05:27:08.821817|COPP_TABLE:queue4_group3|SET|cbs:100|cir:100|meter_type:packets|mode:sr_tcm|queue:4|red_action:drop|trap_action:trap|trap_priority:4|trap_ids:lldp,udld
2026-04-20.05:27:08.821882|COPP_TABLE:queue1_group1|SET|cbs:6000|cir:6000|meter_type:packets|mode:sr_tcm|queue:1|red_action:drop|trap_action:trap|trap_priority:1|trap_ids:ip2me
2026-04-20.05:27:08.821889|COPP_TABLE:queue4_group1|SET|cbs:6000|cir:6000|meter_type:packets|mode:sr_tcm|queue:4|red_action:drop|trap_action:trap|trap_priority:4|trap_ids:bgp,bgpv6,lacp
2026-04-20.05:27:08.821915|COPP_TABLE:queue4_group2|SET|cbs:600|cir:600|meter_type:packets|mode:sr_tcm|queue:4|red_action:drop|trap_action:copy|trap_priority:4|trap_ids:arp_req,arp_resp,neigh_discovery
2026-04-20.05:27:08.821922|COPP_TABLE:default|SET|cbs:600|cir:600|meter_type:packets|mode:sr_tcm|queue:0|red_action:drop|trap_action:trap
```

You may see **duplicate** `SET` lines after restarts or reprogramming (second block in the capture around **06:05**); that is normal when SWSS replays policy.

---

### Verification Step 5 — STATE_DB `COPP_GROUP_TABLE` (operational state)

**Why:** **STATE_DB** exposes **orchestration / hardware sync** status for CoPP groups. A healthy **`state`** (commonly **`ok`**) is a **binary pass/fail** gate before you conclude verification or escalate to **syncd** / **SAI** logs.

```bash
redis-cli -n 6 hgetall "COPP_GROUP_TABLE|queue4_group1"
```

**Expected shape (on a healthy device):**

```text
1) "state"
2) "ok"
```

If **`state`** is not **`ok`**, use **Troubleshooting** (below) before signing off verification or opening a **CoPP policy** change.

---

### Verification Step 6 — (Out of scope for read-only run) Policy change workflow

**Why:** Live networks sometimes require **raising** policer headroom (for example sustained BGP update volume) or **splitting** trap groups. **Wrong** CoPP changes can **starve** **LACP**, **BGP**, or **ARP**—this step is **not** part of read-only verification; it belongs in a **change record** with rollback.

High-level patterns (follow your **release documentation** and **Cisco SONiC** operational guide for **`config`** / **CONFIG_DB**):

1. Model changes in **CONFIG_DB** (or the supported **`config`** CoPP path for your branch), not only by editing **`copp_cfg.json`** on disk unless your process explicitly merges seed files on boot.
2. Apply via **`config load`** / **`config reload`** (or vendor-approved procedure); expect **SWSS** / **syncd** interaction and possible **brief** control-plane risk.
3. **Re-run Verification Steps 1–5** and **protocol-level** validation (BGP sessions, LACP neighbors, ARP/ND completeness).

Editing **`/etc/sonic/copp_cfg.json`** alone **does not** reliably alter runtime policy on most images; assume **reload** or **swss** restart until proven otherwise for your build.

---

### Troubleshooting (short)

| Symptom | Checks |
|---------|--------|
| No **`COPP_TABLE`** rows in APPL_DB | **`show feature status`**, **`supervisorctl status`** inside **swss**, **`grep -i copp swss.rec`**, CONFIG_DB merge errors in **`syslog`**. |
| **`state` not `ok`** in **STATE_DB** | **syncd** / SAI errors in **`/var/log/syslog`** or **`docker logs syncd`**; ASIC resource exhaustion (rare) on heavy custom trap lists. |
| BGP flaps under load | Compare **`queue4_group1`** CIR/CBS with measured **pps** to CPU; any **CIR/CBS** increase follows **change management**, baseline capture, and **post-change** Verification Steps 3–5. |
| ARP/ND issues | Note **`queue4_group2`** uses **`trap_action: copy`** in the seed—verify neighbor learning with **ASIC**/**kernel** tools per TAC guidance if behavior differs from expectation. |

---

### CoPP verification checklist

Complete in order during **standards-based** CoPP health checks or **post-maintenance** validation.

- [ ] **Verification Step 1:** **`cat /etc/sonic/copp_cfg.json`** documents expected **`COPP_GROUP`** / **`COPP_TRAP`** names, **CIR/CBS**, and **queues** for the image under audit.
- [ ] **Verification Step 2:** **`show feature`** shows **swss** **enabled**; **`docker exec -it swss bash`** plus **`ps`** / **`supervisorctl status`** show **orchagent** and **coppmgrd** **RUNNING**.
- [ ] **Verification Step 3:** **APPL_DB** **`COPP_TABLE:<group>`** hashes match **seed** intent for **`queue4_group1`**, **`queue4_group2`**, **`queue4_group3`**, **`queue1_group1`**, and **`default`** (merged **`trap_ids`**, **cir**, **cbs**, **trap_action**).
- [ ] **Verification Step 4:** **`grep -i copp /var/log/swss/swss.rec`** shows **`COPP_TABLE:*|SET|...`** lines **consistent** with **Verification Step 3** after the last **reload** / **swss** event.
- [ ] **Verification Step 5:** **STATE_DB** **`COPP_GROUP_TABLE|<group>`** reports healthy **`state`** (commonly **`ok`**) for representative groups; investigate before signing off if not.

---

## Appendix — CoPP vs L3 ACL (operational distinction)

| Topic | CoPP | L3 ACL (ingress/egress port ACL) |
|--------|------|-----------------------------------|
| Purpose | **Protect the CPU** from **trap/copy** control-plane and punt classes (**BGP**, **ARP**, **ip2me**, **sFlow** samples, etc.). | **Filter user/data-plane** traffic on **front-panel** interfaces per match criteria. |
| Typical tables | **`COPP_GROUP`**, **`COPP_TRAP`**, APPL **`COPP_TABLE`**. | **`ACL_TABLE`**, **`ACL_RULE`** (Config DB / APPL_DB per feature). |
| Policy seed | **`/etc/sonic/copp_cfg.json`** (merged with CONFIG_DB). | JSON fragments / **`config load`** (image-dependent). |

---

*Exercise 1 ACL outputs were transcribed from a lab capture (packet/byte counts vary with traffic). Exercise 2 CoPP excerpts reflect Cisco 8000 SONiC validation. Redis DB indices, ACL IDs, PIDs, and timestamps differ by image—confirm on your target system.*
