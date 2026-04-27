# SONiC Training Lab — Getting Started

Welcome! This guide walks you through connecting to your lab environment and
getting comfortable with the tools you'll use throughout the day.

---

## Your Lab Details

Your instructor will give you the following at the start of the session:

| Detail | Value |
|---|---|
| Server IP | `10.29.209.5` |
| SSH Port | `<YOUR_PORT>` |
| Username | `<Your CCO username>` |
| Password | `cisco123` |

---

## Step 1 — Connect to Your Container

You have a personal Linux container running on the lab server and which you'll use as a jumphost to reach your SONiC nodes. 

Connect to it using SSH from your laptop:

```bash
ssh -p <YOUR_PORT> <YOUR_USERNAME>@10.29.209.5
```

**Example** (port 2203, username `suhahmad`):
```bash
ssh -p 2203 suhahmad@10.29.209.5
```

> **Windows users:** Use [PuTTY](https://www.putty.org/) or the built-in
> Windows Terminal / PowerShell — they all support `ssh` natively.

---

## Step 2 — Explore Your Environment

Your **SSH config** has also been pre-populated, so you can reach any device
in your pod by short name:

```bash
# These all work — pick whichever is easier to type
ssh leaf1
ssh leaf2
ssh leaf3
ssh spine4
```

```diff
nicmcl@16e461144464:~$ ssh leaf1
Warning: Permanently added '10.29.209.7' (ED25519) to the list of known hosts.
Warning: Permanently added '192.168.122.51' (RSA) to the list of known hosts.
Debian GNU/Linux 12 \n \l

+ credentials are "admin/password" for the SONiC devices.
+ admin@192.168.122.51's password: 

Linux pod9-leaf1 6.1.0-11-2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.38-4 (2023-08-08) x86_64
You are on
  ____   ___  _   _ _  ____
 / ___| / _ \| \ | (_)/ ___|
 \___ \| | | |  \| | | |
  ___) | |_| | |\  | | |___
 |____/ \___/|_| \_|_|\____|

 Software for Open Networking in the Cloud

Unauthorized access and/or use are prohibited.
All access and/or use are subject to monitoring.

Help:    https://sonic-net.github.io/SONiC/

Last login: Mon Apr 20 06:05:51 2026 from 192.168.122.1
admin@pod9-leaf1:~$

```

---

## Step 3 — Verify Connectivity

SSH directly into any device in your pod using the short names from your SSH config:

```bash
# Connect to Leaf1
ssh leaf1

# Once inside SONiC — check the version
show version

# Check interface status
show interfaces status

# Exit back to your container
exit
```

Repeat for any other device:

```bash
ssh leaf2
ssh leaf3
ssh spine4
```

Device password: **`password`**

---

## Troubleshooting

**Can't SSH into the container?**
- Double-check the port number and server IP with your instructor.
- Make sure you're using the correct username (case-sensitive).

**Can't reach a SONiC device?**
- The devices may still be booting — give them a few minutes after the
  instructor starts the lab.
- Try `ssh leaf1` and check if the connection times out or is refused.
- Ask your instructor to verify the pod status.
    podN-spine4  ansible_host=192.168.122.39

---

## Getting Help

Raise your hand or message the instructor in the chat. Don't struggle
silently — we're here to help!

**Please proceed to Lab_1**: [LINK](../lab_1/lab_1_guide.md)
