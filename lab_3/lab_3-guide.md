# Lab 3 Guide: SONiC Automation on Cisco 8000 [45 Min]

---

## Introduction

SONiC runs on Debian Linux and exposes multiple configuration interfaces: a Python Click-based CLI, JSON file loading, configuration patches, gNMI, and REST APIs. This flexibility makes SONiC a natural fit for network automation. In this lab you will explore two automation approaches: Ansible for multi-device orchestration and gNMI for programmatic device access.

---

## Table of Contents

- [Lab 3 Guide: SONiC Automation on Cisco 8000 \[45 Min\]](#lab-3-guide-sonic-automation-on-cisco-8000-45-min)
  - [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Lab Objectives](#lab-objectives)
  - [Lab Environment](#lab-environment)
    - [Credentials](#credentials)
    - [gNMI](#gnmi)
  - [Task 1 — Ansible Automation](#task-1--ansible-automation)
    - [1.1 Ansible Inventory](#11-ansible-inventory)
    - [1.2 SONiC CLI via `ansible.builtin.shell`](#12-sonic-cli-via-ansiblebuiltinshell)
    - [1.3 The `cisco.sonic` Ansible Collection](#13-the-ciscosonic-ansible-collection)
      - [Key Features of `cisco.sonic`](#key-features-of-ciscosonic)
    - [1.4 Ansible Summary](#14-ansible-summary)
  - [Task 2 — gNMI Automation](#task-2--gnmi-automation)
    - [2.1 Enabling gNMI on SONiC](#21-enabling-gnmi-on-sonic)
      - [GNMI CONFIG\_DB Schema](#gnmi-config_db-schema)
      - [Practical Example — Lab Configuration](#practical-example--lab-configuration)
    - [2.2 gNMI Capabilities](#22-gnmi-capabilities)
    - [2.3 SONiC DB GET — Querying CONFIG\_DB Directly](#23-sonic-db-get--querying-config_db-directly)
    - [2.4 SONiC Native YANG GET](#24-sonic-native-yang-get)
    - [2.5 OpenConfig GET](#25-openconfig-get)
    - [2.6 Comparing the Three gNMI Models](#26-comparing-the-three-gnmi-models)
    - [2.7 gNMI SET — Making Configuration Changes](#27-gnmi-set--making-configuration-changes)
    - [2.8 gNMI SUBSCRIBE — Streaming Telemetry](#28-gnmi-subscribe--streaming-telemetry)
  - [End of Lab](#end-of-lab)
4. [End of Lab](#end-of-lab)

---

## Lab Objectives

By completing this lab you will be able to:

- Build an Ansible inventory for SONiC devices using SSH aliases
- Write Ansible playbooks that configure SONiC using CLI commands via `ansible.builtin.shell`
- Use the `cisco.sonic` Ansible collection for structured SONiC and FRR operations
- Understand how gNMI is enabled on a SONiC device
- Identify the three gNMI data models: SONiC DB, SONiC Native YANG, and OpenConfig
- Read and interpret gNMI GET responses across all three models
- Understand gNMI SET for pushing configuration changes
- Subscribe to streaming telemetry with gNMI SUBSCRIBE

---

## Lab Environment

You are working from a **Linux container** that has SSH connectivity to all devices in your pod. SSH aliases are pre-configured so you can connect by name:

```bash
ssh leaf1
ssh leaf2
ssh leaf3
ssh spine4
```

The container has `ansible` and `python3` pre-installed.

### Credentials

| Field    | Value    |
|----------|----------|
| Username | admin    |
| Password | password |

### gNMI

The gNMI commands in Task 2 use `gnmic`, which is installed directly on **Leaf1** for ease of access. The trainer will demonstrate these commands live. The commands and expected outputs are included in this guide as reference.

---

## Task 1 — Ansible Automation

> **Goal:** Use Ansible to push configuration to SONiC devices from your container. We will explore two approaches — built-in shell commands and the `cisco.sonic` Ansible collection — using VLAN configuration as a running example.
>
> All Ansible commands are run from your **container**.

---

### 1.1 Ansible Inventory

Before writing playbooks, set up the inventory file that tells Ansible how to reach each device.

**Run** — Create the project directory:

```bash
mkdir -p ~/sonic-automation
cd ~/sonic-automation
```

Create `~/sonic-automation/inventory.yaml` with the following content using vi from the cli:

```yaml
leaf_switches:
  hosts:
    leaf1:
  vars:
    ansible_user: admin
    ansible_ssh_pass: password
    ansible_become_password: password
```

**Run** — Test connectivity:

```bash
ansible all -i inventory.yaml -m ping
```

```
leaf1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

### 1.2 SONiC CLI via `ansible.builtin.shell`

The simplest approach: run SONiC CLI commands directly through an SSH shell. This is the same as if you SSH'd into the device and typed the commands yourself.

Create `~/sonic-automation/vlan_cli.yaml` with the following content:

```yaml
---
- name: Configure VLANs using SONiC CLI
  gather_facts: false
  hosts: leaf_switches
  tasks:
    - name: Create VLAN 100
      ansible.builtin.shell: config vlan add 100
      become: true

    - name: Create VLAN 200
      ansible.builtin.shell: config vlan add 200
      become: true

    - name: Add Ethernet40 to VLAN 100 as untagged
      ansible.builtin.shell: config vlan member add 100 -u Ethernet40
      become: true

    - name: Add Ethernet44 to VLAN 200 as untagged
      ansible.builtin.shell: config vlan member add 200 -u Ethernet44
      become: true

    - name: Save configuration
      ansible.builtin.shell: config save -y
      become: true
```
> [!Note]
> The syntax *`become: true`* instructs Ansible to execute the command with `sudo`

**Run** — Execute the playbook:

```bash
ansible-playbook -i inventory.yaml vlan_cli.yaml
```

```
PLAY [Configure VLANs using SONiC CLI] ****************************************

TASK [Create VLAN 100] ********************************************************
changed: [leaf1]

TASK [Create VLAN 200] ********************************************************
changed: [leaf1]

...

PLAY RECAP ********************************************************************
leaf1    : ok=5    changed=5    unreachable=0    failed=0
```

**Verify** — SSH into Leaf1 and check the VLANs:

```bash
ssh leaf1
```
```bash
show vlan brief
```

```
+-----------+--------------+------------+----------------+-------------+
| VLAN ID   | IP Address   | Ports      | Port Tagging   | Proxy ARP   |
+===========+==============+============+================+=============+
| 100       |              | Ethernet40 | untagged       | disabled    |
+-----------+--------------+------------+----------------+-------------+
| 200       |              | Ethernet44 | untagged       | disabled    |
+-----------+--------------+------------+----------------+-------------+
```

> **Pros:** Simple, familiar, no extra modules needed.

> **Cons:** No idempotency — running the playbook again may produce errors if VLANs already exist. No structured error handling.

---

### 1.3 The `cisco.sonic` Ansible Collection

> [!Ansible-Galaxy-Reference]
> https://galaxy.ansible.com/ui/repo/published/cisco/sonic/

The `cisco.sonic` collection provides purpose-built modules for SONiC that support structured error handling, conditional execution (`wait_for`), and atomic rollback (`all_or_none`).

Since the VLANs are already configured from the previous playbook, we will use the collection to **verify** the existing configuration and **configure FRR routing** — tasks that highlight the collection's unique strengths.

The `cisco.sonic` modules use `network_cli` (SSH-based CLI sessions), so we need to add connection variables to our inventory. Update `~/sonic-automation/inventory.yaml` and append these two lines at the end:

```yaml
    ansible_connection: network_cli
    ansible_network_os: cisco.sonic.sonic
```

After editing `inventory.yaml` should look like:

```yaml
leaf_switches:
  hosts:
    leaf1:
  vars:
    ansible_user: admin
    ansible_ssh_pass: password
    ansible_become_password: password
    ansible_connection: network_cli
    ansible_network_os: cisco.sonic.sonic
```

**export proxy** - Prior to installing the Ansible cisco.sonic collection we'll need to set the http(s) proxy on our pods:
```bash
export http_proxy="http://proxy-wsa.esl.cisco.com:80"
export https_proxy="http://proxy-wsa.esl.cisco.com:80"
```

**Run** — Install the collection (if not already installed):

```bash
ansible-galaxy collection install cisco.sonic
```

Create `~/sonic-automation/verify_and_configure.yaml` with the following content:

```yaml
---
- name: Verify VLANs and configure BGP
  hosts: leaf_switches
  gather_facts: false
  connection: network_cli
  tasks:
    - name: Verify VLANs 100 and 200 exist
      cisco.sonic.sonic_command:
        commands:
          - "show vlan brief"
        wait_for:
          - result[0] contains 100
          - result[0] contains 200
        match: all
        retries: 3
        interval: 5
      register: vlan_output

    - name: Display VLAN status
      ansible.builtin.debug:
        msg: "{{ vlan_output.stdout_lines }}"

    - name: Show BGP summary
      cisco.sonic.sonic_frr_command:
        commands:
          - "show bgp summary"
      register: bgp_output

    - name: Display BGP status
      ansible.builtin.debug:
        msg: "{{ bgp_output.stdout_lines }}"

    - name: Create a prefix-list in FRR
      cisco.sonic.sonic_frr_config:
        lines:
          - "ip prefix-list DENY_DEFAULT seq 10 deny 0.0.0.0/0"
          - "ip prefix-list DENY_DEFAULT seq 20 permit 0.0.0.0/0 le 32"
```

**Run** — Execute the playbook:

```bash
ansible-playbook -i inventory.yaml verify_and_configure.yaml
```

#### Key Features of `cisco.sonic`

| Feature | Description |
|---------|-------------|
| `sonic_command` | Run SONiC show commands with conditional checks |
| `sonic_config` | Push SONiC CLI config with `all_or_none` rollback |
| `sonic_frr_command` | Run FRR show commands (`show bgp summary`, `show route`, etc.) |
| `sonic_frr_config` | Push FRR config with hierarchical `parents` context |
| `wait_for` | Polls command output until a condition is met or retries are exhausted |
| `match: any` / `match: all` | Requires any or all `wait_for` conditions to pass |
| `all_or_none: 'none'` | Rolls back all commands if any command fails |
| `parents` | Specifies the parent context for hierarchical configs (e.g., `router bgp 65000`) |

---

### 1.4 Ansible Summary

| Method | Best For | Idempotent? | Error Handling |
|--------|----------|-------------|----------------|
| `ansible.builtin.shell` | Quick tasks, prototyping | No | Manual (check `rc`) |
| `cisco.sonic` collection | Production automation, CI/CD pipelines | Via `wait_for` | `all_or_none`, `wait_for`, `retries` |

---

## Task 2 — gNMI Automation

> **Goal:** Understand how gNMI works on SONiC — how to enable it, the three supported data models, and how to query and configure device state programmatically.
>
> The gNMI commands below will be **demonstrated by the trainer** on Leaf1. Follow along with the commands and expected outputs provided here.

gNMI (gRPC Network Management Interface) is a protocol for network device configuration and telemetry based on gRPC. SONiC supports gNMI through the `gnmi` container.

SONiC's gNMI server supports three types of data models:

| Model Type | Path Style | Example |
|-----------|------------|---------|
| **SONiC DB** (sonic-db) | `TABLE/KEY` targeting a specific database | `--path PORT/Ethernet0 --target CONFIG_DB` |
| **SONiC Native YANG** | `sonic-module:container/list[key=value]` | `sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]` |
| **OpenConfig** | `openconfig-module:container/list[key=value]` | `openconfig-interfaces:interfaces/interface[name=Ethernet0]` |

> **Important:** gNMI support is determined at **image build time**. The SONiC build system uses flags like `INCLUDE_SYSTEM_GNMI`, `ENABLE_TRANSLIB_WRITE`, and `ENABLE_NATIVE_WRITE` in `rules/config` to control whether the gNMI container is included and which write operations are permitted. The YANG models available for gNMI queries (OpenConfig, SONiC native YANG) are also packaged during the build. Work with your Cisco team to ensure your image has the appropriate gNMI and model support.

---

### 2.1 Enabling gNMI on SONiC

Before using gNMI, the gNMI server must be configured on the device. SONiC's gNMI server is managed through the `GNMI` table in CONFIG_DB, which has two keys: `GNMI|gnmi` for server settings and `GNMI|certs` for TLS certificates.

#### GNMI CONFIG_DB Schema

The full configuration schema with all available fields:

```
{
    "GNMI": {
        "gnmi": {
            "port": "<port-number>",
            "client_auth": "true|false",
            "log_level": "<0-7>",
            "threshold": "<max-connections>",
            "idle_conn_duration": "<seconds>",
            "save_on_set": "true|false",
            "user_auth": "password|cert",
            "enable_crl": "true|false",
            "crl_expire_duration": "<seconds>"
        },
        "certs": {
            "server_crt": "<path-to-server-cert>",
            "server_key": "<path-to-server-key>",
            "ca_crt": "<path-to-ca-cert>"
        }
    }
}
```

**`GNMI|gnmi` — Server settings:**

| Field | Description |
|-------|-------------|
| `port` | TCP port the gNMI server listens on |
| `client_auth` | When `false`, client certificates are not required |
| `log_level` | Logging verbosity (higher = more verbose) |
| `threshold` | Maximum number of concurrent connections handled sequentially |
| `idle_conn_duration` | Seconds before the server closes idle client connections |
| `save_on_set` | When `true`, automatically runs `config save` after every SET operation |
| `user_auth` | Authentication mode: `password` (username/password via gRPC metadata) or `cert` (client certificate). When set to `cert`, the server reads allowed certificates from the `GNMI_CLIENT_CERT` CONFIG_DB table |
| `enable_crl` | Enable certificate revocation list checking (only when `user_auth` is `cert`) |
| `crl_expire_duration` | CRL cache expiry in seconds |

**`GNMI|certs` — TLS certificate paths:**

| Field | Description |
|-------|-------------|
| `server_crt` | Path to the server TLS certificate |
| `server_key` | Path to the server TLS private key |
| `ca_crt` | Path to the CA certificate for client certificate validation |

> **TLS behavior:** If the `certs` key is absent (or `server_crt`/`server_key` are empty), the gNMI server starts in **non-TLS mode** (`--noTLS`). This is convenient for labs but should never be used in production.

#### Practical Example — Lab Configuration

In our lab we use non-TLS mode with password authentication. By omitting the `certs` section entirely, the server starts without TLS.

**Run** — SSH into Leaf1:

```bash
ssh leaf1
```

Using a text editor, create `/tmp/gnmi.json` with the following content:

```json
{
    "GNMI": {
        "gnmi": {
            "client_auth": "false",
            "log_level": "2",
            "port": "50051"
        }
    }
}
```

**Run** — Load the configuration into CONFIG_DB:

```bash
sudo config load /tmp/gnmi.json -y
```

```
Running command: /usr/local/bin/sonic-cfggen -j /tmp/gnmi.json --write-to-db
```

**Run** — Restart the gnmi container to apply the changes:

```bash
sudo systemctl restart gnmi
```

**Verify** — Check that the gnmi container is running:

```bash
docker ps -f name=gnmi
```

You should see the `gnmi` container with status `Up`.

**Verify** — Confirm the gNMI server is listening on port 50051:

```bash
sudo ss -tlnp | grep 50051
```

```
LISTEN  0  4096  *:50051  *:*  users:(("telemetry",pid=xxxx,fd=xx))
```

This confirms the gNMI server process is bound and listening on port 50051.

**Run** — Save the configuration so it persists across reboots:

```bash
sudo config save -y
```

> **In our lab**, your container cannot directly reach the gNMI port (50051) on the SONiC devices. The walkthrough above is provided so you understand how gNMI is configured. The instructor will demonstrate gNMI capabilities live on a device.

---

### 2.2 gNMI Capabilities

The Capabilities RPC tells you which YANG models the device supports, the gNMI version, and the supported encodings.

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  capabilities
```

```
gNMI version: 0.7.0
supported models:
  - openconfig-acl, OpenConfig working group, 1.0.2
  - openconfig-interfaces, OpenConfig working group,
  - openconfig-platform, OpenConfig working group,
  - openconfig-network-instance, OpenConfig working group,
  - openconfig-routing-policy, OpenConfig working group,
  - openconfig-sampling-sflow, OpenConfig working group,
  - openconfig-mclag, OpenConfig working group,
  - openconfig-system, OpenConfig working group,
  - openconfig-lldp, OpenConfig working group, 1.0.2
  - openconfig-lldp-ext, SONiC, 0.1.0
  - ietf-yang-library, IETF NETCONF (Network Configuration) Working Group, 2016-06-21
  - sonic-db, SONiC, 0.1.0
supported encodings:
  - JSON
  - JSON_IETF
  - PROTO
```

Notice the `sonic-db` model at the bottom — this is a special SONiC model that lets you query Redis databases directly using their native schema.

---

### 2.3 SONiC DB GET — Querying CONFIG_DB Directly

The `sonic-db` model allows you to query any SONiC Redis database (CONFIG_DB, APPL_DB, STATE_DB, etc.) using the database's native key structure. This is equivalent to running `redis-cli` commands but over gNMI.

**GET a port entry from CONFIG_DB:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path PORT/Ethernet0 \
  --target CONFIG_DB
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777345110487030108,
    "time": "2026-04-27T19:58:30.487030108-07:00",
    "target": "CONFIG_DB",
    "updates": [
      {
        "Path": "PORT/Ethernet0",
        "values": {
          "PORT/Ethernet0": {
            "admin_status": "up",
            "alias": "etp0",
            "index": "0",
            "lanes": "2304,2305,2306,2307",
            "mtu": "9100",
            "speed": "100000"
          }
        }
      }
    ]
  }
]
```

This returns exactly what you would see by running `sonic-db-cli CONFIG_DB hgetall 'PORT|Ethernet0'` — but through a standard gNMI interface. Notice all values are strings — this is a characteristic of the Redis-backed sonic-db model.

**GET VLAN configuration:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path VLAN \
  --target CONFIG_DB
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777346234185824264,
    "time": "2026-04-27T20:17:14.185824264-07:00",
    "target": "CONFIG_DB",
    "updates": [
      {
        "Path": "VLAN",
        "values": {
          "VLAN": {
            "Vlan100": {
              "vlanid": "100"
            }
          }
        }
      }
    ]
  }
]
```

Using just `VLAN` as the path returns all VLANs. You can narrow it to a specific VLAN with `VLAN/Vlan100`.

**GET from other databases:**

You can target any SONiC database by changing `--target`:

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path PORT_TABLE/Ethernet0 \
  --target APPL_DB
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777346235106918580,
    "time": "2026-04-27T20:17:15.10691858-07:00",
    "target": "APPL_DB",
    "updates": [
      {
        "Path": "PORT_TABLE/Ethernet0",
        "values": {
          "PORT_TABLE/Ethernet0": {
            "admin_status": "up",
            "alias": "etp0a",
            "index": "0",
            "lanes": "2304,2305",
            "mtu": "9100",
            "oper_status": "down",
            "speed": "100000"
          }
        }
      }
    ]
  }
]
```

Notice APPL_DB includes `oper_status` — this is the application-level state that orchagent uses to program the ASIC, as opposed to CONFIG_DB which only stores the desired configuration.

**GET interface counters from COUNTERS_DB (virtual paths):**

COUNTERS_DB stores raw SAI counters keyed by OID (e.g., `oid:0x100000000000f`). The sonic-db gNMI model supports **virtual paths** that let you use the interface name instead, and drill down to a specific counter:

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS \
  --target COUNTERS_DB
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777346421140381531,
    "time": "2026-04-27T20:20:21.140381531-07:00",
    "target": "COUNTERS_DB",
    "updates": [
      {
        "Path": "COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS",
        "values": {
          "COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS": "0"
        }
      }
    ]
  }
]
```

The gNMI server automatically resolves `Ethernet0` to its OID via the `COUNTERS_PORT_NAME_MAP` table — you never need to look up OIDs manually.

Virtual paths also support wildcards. To get a single counter across all ports:

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path "COUNTERS/Ethernet*/SAI_PORT_STAT_IF_IN_OCTETS" \
  --target COUNTERS_DB
```

This returns `SAI_PORT_STAT_IF_IN_OCTETS` for every Ethernet port on the device in a single response — useful for building per-port traffic dashboards.

The supported virtual path patterns for COUNTERS_DB:

| Virtual Path | Returns |
|-------------|---------|
| `COUNTERS/Ethernet0` | All counters for one port |
| `COUNTERS/Ethernet0/<counter_name>` | One specific counter for one port |
| `COUNTERS/Ethernet*` | All counters for all ports |
| `COUNTERS/Ethernet*/<counter_name>` | One counter across all ports |

> **Key takeaway:** The `--target` flag selects which Redis database to query. The `--path` follows the Redis key structure: `TABLE/KEY` or just `TABLE` to get all entries. COUNTERS_DB supports virtual paths that resolve interface names to OIDs and allow drilling down to specific counters.

---

### 2.4 SONiC Native YANG GET

SONiC also publishes native YANG models that map to its configuration database schema. These provide a more structured, standards-compliant interface compared to raw database access.

The native YANG models are maintained at: https://github.com/sonic-net/sonic-buildimage/tree/master/src/sonic-yang-models/yang-models

**GET a port using native YANG:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777345111194308550,
    "time": "2026-04-27T19:58:31.19430855-07:00",
    "updates": [
      {
        "Path": "sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]",
        "values": {
          "sonic-port:sonic-port/PORT/PORT_LIST": {
            "sonic-port:PORT_LIST": [
              {
                "admin_status": "up",
                "alias": "etp0",
                "index": 0,
                "lanes": "2304,2305,2306,2307",
                "mtu": 9100,
                "name": "Ethernet0",
                "speed": 100000,
                "subport": 0
              }
            ]
          }
        }
      }
    ]
  }
]
```

Notice the difference from the sonic-db response: the native YANG model returns typed values (integers for `mtu`, `speed`, `index`) rather than strings, and uses a structured path with module prefixes.

---

### 2.5 OpenConfig GET

OpenConfig models provide a vendor-neutral view of device configuration and state. SONiC translates between its internal CONFIG_DB/STATE_DB and the OpenConfig schema.

**GET an interface using OpenConfig:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path openconfig-interfaces:interfaces/interface[name=Ethernet0]
```

Output (abbreviated):
```json
[
  {
    "source": "1.18.1.8:50051",
    "timestamp": 1777345112014496593,
    "time": "2026-04-27T19:58:32.014496593-07:00",
    "updates": [
      {
        "Path": "openconfig-interfaces:interfaces/interface[name=Ethernet0]",
        "values": {
          "openconfig-interfaces:interfaces/interface": {
            "openconfig-interfaces:interface": [
              {
                "config": {
                  "enabled": true,
                  "mtu": 9100,
                  "name": "Ethernet0",
                  "type": "iana-if-type:ethernetCsmacd"
                },
                "name": "Ethernet0",
                "openconfig-if-ethernet:ethernet": {
                  "config": {
                    "port-speed": "openconfig-if-ethernet:SPEED_100GB"
                  },
                  "state": {
                    "port-speed": "openconfig-if-ethernet:SPEED_100GB"
                  }
                },
                "state": {
                  "admin-status": "UP",
                  "counters": {
                    "in-broadcast-pkts": "0",
                    "in-discards": "0",
                    "in-errors": "0",
                    "in-octets": "0",
                    "in-pkts": "0",
                    "out-broadcast-pkts": "0",
                    "out-discards": "0",
                    "out-errors": "0",
                    "out-octets": "0",
                    "out-pkts": "0"
                  },
                  "enabled": true,
                  "mtu": 9100,
                  "name": "Ethernet0",
                  "oper-status": "DOWN",
                  "type": "iana-if-type:ethernetCsmacd"
                }
              }
            ]
          }
        }
      }
    ]
  }
]
```

OpenConfig responses include both `config` (desired state) and `state` (actual operational state including counters). This is richer than the sonic-db response, which only returns CONFIG_DB contents.

---

### 2.6 Comparing the Three gNMI Models

All three approaches query the same device — the difference is in the data model used:

| Aspect | SONiC DB (`sonic-db`) | SONiC Native YANG | OpenConfig |
|--------|----------------------|-------------------|------------|
| **Path style** | `TABLE/KEY --target DB` | `sonic-module:path/LIST[key=val]` | `oc-module:path/list[key=val]` |
| **Schema source** | Redis DB schema | SONiC YANG models | OpenConfig YANG models |
| **Data types** | All strings (Redis) | Typed (int, string, enum) | Typed with OC enumerations |
| **Config vs State** | Depends on target DB | Config only (native models) | Both config and state |
| **Vendor neutral** | No (SONiC-specific) | No (SONiC-specific) | Yes |
| **Best for** | Direct DB queries, debugging | SONiC-native automation | Multi-vendor automation |

---

### 2.7 gNMI SET — Making Configuration Changes

gNMI is not just for reading — you can also push configuration changes using the SET RPC. SET uses the OpenConfig model and requires `json_ietf` encoding. The path targets the config container, and the value wraps the field inside the module-prefixed container name.

**Update MTU to 1500 via OpenConfig:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  set \
  --update-path /openconfig-interfaces:interfaces/interface[name=Ethernet0]/config \
  --update-value '{"openconfig-interfaces:config": {"mtu": 1500}}' \
  --encoding json_ietf
```

```json
{
  "source": "1.18.1.8:50051",
  "time": "1969-12-31T16:00:00-08:00",
  "results": [
    {
      "operation": "UPDATE",
      "path": "openconfig-interfaces:interfaces/interface[name=Ethernet0]/config"
    }
  ]
}
```

**Verify** — Confirm the change via sonic-db GET:

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path PORT/Ethernet0 \
  --target CONFIG_DB
```

```json
[
  {
    "source": "1.18.1.8:50051",
    "target": "CONFIG_DB",
    "updates": [
      {
        "Path": "PORT/Ethernet0",
        "values": {
          "PORT/Ethernet0": {
            "admin_status": "up",
            "alias": "etp0",
            "index": "0",
            "lanes": "2304,2305,2306,2307",
            "mtu": "1500",
            "speed": "100000"
          }
        }
      }
    ]
  }
]
```

The `mtu` field now shows `"1500"` — confirming the SET took effect in CONFIG_DB.

**Revert MTU back to 9100:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  set \
  --update-path /openconfig-interfaces:interfaces/interface[name=Ethernet0]/config \
  --update-value '{"openconfig-interfaces:config": {"mtu": 9100}}' \
  --encoding json_ietf
```

> **Important:** gNMI SET writes directly to CONFIG_DB. The change takes effect immediately but is not automatically saved to `/etc/sonic/config_db.json`. To persist, run `sudo config save -y` on the device — or set `"save_on_set": "true"` in the `GNMI|gnmi` configuration to have every SET automatically save.

---

### 2.8 gNMI SUBSCRIBE — Streaming Telemetry

The SUBSCRIBE RPC opens a long-lived stream from the device. This is how gNMI delivers real-time telemetry — the device pushes updates to the collector instead of being polled.

**Subscribe to a specific interface counter:**

```bash
gnmic -a 1.18.1.8:50051 \
  -u admin -p password \
  --insecure \
  subscribe \
  --path "COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS" \
  --target COUNTERS_DB \
  --stream-mode sample \
  --sample-interval 5s
```

The device pushes the current value immediately, then continues streaming at the requested interval:

```json
{
  "source": "1.18.1.8:50051",
  "subscription-name": "default-1777347176",
  "timestamp": 1777347118691257489,
  "time": "2026-04-27T20:31:58.691257489-07:00",
  "target": "COUNTERS_DB",
  "updates": [
    {
      "Path": "COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS",
      "values": {
        "COUNTERS/Ethernet0/SAI_PORT_STAT_IF_IN_OCTETS": "0"
      }
    }
  ]
}
```

Press `Ctrl+C` to stop the subscription.

SUBSCRIBE works with the sonic-db targets (`CONFIG_DB`, `APPL_DB`, `COUNTERS_DB`) and `OC_YANG` for OpenConfig paths. In production, collectors like Telegraf or gNMI pipeline ingest these updates continuously for monitoring dashboards and alerting.

---

## End of Lab

This lab is complete. You have:

- Built an Ansible inventory for SONiC using SSH aliases and YAML format
- Written and executed a playbook using `ansible.builtin.shell` to configure VLANs on Leaf1
- Used the `cisco.sonic` Ansible collection to verify VLANs, query BGP, and configure FRR prefix-lists
- Walked through enabling gNMI on a SONiC device (configuration, restart, port verification)
- Explored gNMI Capabilities to discover supported YANG models
- Queried SONiC using all three gNMI data models: SONiC DB, SONiC Native YANG, and OpenConfig
- Pushed configuration changes via gNMI SET using OpenConfig
- Subscribed to streaming telemetry with gNMI SUBSCRIBE

**Ansible** provides two methods of increasing sophistication:

| Method | Module | Mechanism |
|--------|--------|-----------|
| Shell commands | `ansible.builtin.shell` | Runs SONiC CLI commands via SSH |
| cisco.sonic collection | `cisco.sonic.sonic_config` | Structured CLI over network_cli with error handling |

**gNMI** provides programmatic access through four RPCs:

| RPC | Purpose | Key Detail |
|-----|---------|------------|
| Capabilities | Discover supported models and encodings | Always run first to understand what the device supports |
| GET | Read configuration and state | Three models: sonic-db, native YANG, OpenConfig |
| SET | Push configuration changes | Uses OpenConfig paths with `json_ietf` encoding |
| SUBSCRIBE | Stream real-time telemetry | Device pushes updates to collector; works with sonic-db targets (CONFIG_DB, APPL_DB, COUNTERS_DB) and OC_YANG |

Both Ansible and gNMI can be combined: use Ansible for orchestrating multi-device workflows and gNMI for granular, real-time device interactions.


Proceed to [**Lab 4 (ACL and CoPP)**](../lab_4/lab_4-guide.md) to learn how SONiC implements Access Control Lists and CoPP.