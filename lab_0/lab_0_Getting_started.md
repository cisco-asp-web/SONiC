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

You have a personal Linux container running on the lab server and which you'll use as a jumphost to reach your SONiC nodes. Connect to it using SSH from your laptop:

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

Once logged in you'll find:

```
~/lab/
└── ansible/
    ├── ansible.cfg    # pre-configured defaults
    └── inventory.ini  # your pod's device IPs, ready to use
```

Your **SSH config** has also been pre-populated, so you can reach any device
in your pod by short name:

```bash
# These all work — pick whichever is easier to type
ssh leaf1
ssh leaf2
ssh leaf3
ssh spine4
```

---

## Step 3 — Verify Connectivity

SSH directly into any device in your pod using the short names from your SSH config:

```bash
# Connect to Leaf1
ssh leaf1

# Once inside SONiC — check the version
show version

# show platform to see which kind of device you're running
show platform summary

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

**Ansible returns `unreachable`?**
- Same as above — device may not be ready yet.
- Check `~/lab/ansible/inventory.ini` to confirm the IPs look correct.

```bash
        [leaves]
        podN-leaf1  ansible_host=192.168.122.36
        podN-leaf2  ansible_host=192.168.122.37
        podN-leaf3  ansible_host=192.168.122.38

        [spines]
        podN-spine4  ansible_host=192.168.122.39
```

---

## Getting Help

Raise your hand or message the instructor in the chat. Don't struggle
silently — we're here to help!
