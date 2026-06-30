# Home Lab: Network Segmentation with VLAN-style Isolation and Firewall Rules

## Overview

This project simulates a small home network with a dedicated guest network, isolated from the trusted internal network, using virtual machines and Linux-based routing and firewall rules. It demonstrates core networking and security concepts: subnetting, IP forwarding, routing, and stateful firewall policy — the same principles used in real corporate guest WiFi and VLAN deployments.

I work with production systems and incidents daily, but wanted hands-on experience with the infrastructure layer underneath — specifically how network segmentation and firewall policy are actually implemented, not just diagrammed.

## Architecture

```
                    ┌─────────────────┐
                    │    Internet      │
                    │   (NAT/WAN)      │
                    └────────┬─────────┘
                             │ enp0s3
                    ┌────────┴─────────┐
                    │   router-fw      │
                    │  (Ubuntu 24.04)  │
                    │                  │
                    │  enp0s8   enp0s9 │
                    └────┬────────┬────┘
                         │        │
              192.168.10.0/24  192.168.20.0/24
              (trusted-lan)    (guest-lan)
                         │        │
                ┌────────┴──┐  ┌──┴─────────┐
                │  client-  │  │  client-   │
                │  trusted  │  │  guest     │
                │.10.50     │  │  .20.50    │
                └───────────┘  └────────────┘
```

| Host | Role | IP Address | Network |
|---|---|---|---|
| `router-fw` | Router/Firewall | `192.168.10.1`, `192.168.20.1` | trusted-lan, guest-lan |
| `client-trusted` | Trusted client | `192.168.10.50` | trusted-lan |
| `client-guest` | Guest client | `192.168.20.50` | guest-lan |

## Tools Used

- **VirtualBox** — hypervisor, used to create isolated "Internal Networks" simulating separate VLANs
- **Ubuntu Server 24.04 LTS** — OS for all three VMs
- **Netplan** — static IP configuration
- **iptables** — firewall rule enforcement

## What Was Built

### 1. Network Segmentation
Created two isolated subnets (`192.168.10.0/24` and `192.168.20.0/24`) using VirtualBox Internal Networks, simulating a trusted LAN and a guest LAN as they would exist on separate VLANs in a real switch-based network.

Attachment: 
<img width="353" height="414" alt="firewall static ip" src="https://github.com/user-attachments/assets/ed34e98e-c7cd-453f-96d5-8c8b560f25f0" />

### 2. Routing
Enabled IP forwarding on `router-fw` (`net.ipv4.ip_forward=1`) so it could pass traffic between the two subnets and out to the internet via NAT, acting as the gateway for both networks.

### 3. Firewall Policy
Implemented isolation between guest and trusted networks using `iptables`:

```bash
# Block new connections from guest-lan to trusted-lan
sudo iptables -I FORWARD -i enp0s9 -o enp0s8 -j DROP
```
Attachment: 

<img width="767" height="99" alt="iptables rule success added" src="https://github.com/user-attachments/assets/0581ec24-c762-4348-ac13-2c4b5656249c" />
<img width="531" height="57" alt="ping failed from guest to trust because firewall rules added" src="https://github.com/user-attachments/assets/50fceb3d-8efa-42f2-8651-665d022ecbf6" />

This follows the principle of least privilege used in real guest network design: guest devices are unmanaged and untrusted by default, so they are blocked from initiating contact with internal systems, while still being permitted to reach the internet.

Rules were made persistent across reboots using `iptables-persistent`.

### 4. Verification

| Test | Result | Why |
|---|---|---|
| `client-trusted` → `router-fw` | ✅ Success | Same-subnet, no rule blocks this |
| `client-guest` → `router-fw` | ✅ Success | Guest can still reach its gateway |
| `client-guest` → `client-trusted` | ❌ Blocked (timeout) | DROP rule denies guest-initiated traffic to trusted-lan |
| `client-guest` → Internet (via NAT) | ✅ Success | Rule only targets the guest→trusted path, not internet access |

Verified using `tcpdump` on the receiving interface to confirm whether packets physically arrived or were silently dropped, rather than relying on ping output alone.

## Problems Encountered and Fixed (Troubleshooting Log)

1. **YAML indentation error in Netplan config** — `addresses:` was indented at the wrong level relative to its parent key, causing a "expected mapping, check indentation" error. Fixed by aligning it as a sibling of `dhcp4:` rather than `enp0sX:`.
2. **Duplicate static IP across two interfaces** — both `enp0s8` and `enp0s9` were accidentally assigned `192.168.10.1`, causing inconsistent `ip a` output. Diagnosed by inspecting the raw Netplan file rather than assuming the configuration was correct.
Attachment:

<img width="504" height="323" alt="was failed because es09 was set to 192 168 10 1, which duplicate from trusted client  changed back netplan to 20 1 it worked now" src="https://github.com/user-attachments/assets/f7b7b593-3b83-4677-9663-280f7add6231" />

4. **Asymmetric routing misunderstanding** — initially assumed guest→trusted connectivity was working based on ping behavior, but `tcpdump` revealed packets weren't arriving at all. Root cause: neither client VM had a static route to the other's subnet — each only knew its own directly-connected network. This was a routing gap, not a firewall issue, and was diagnosed by checking `ip route` on both ends rather than assuming the firewall was the only variable.

## Key Concepts Demonstrated

- Subnetting and IP addressing
- IP forwarding / routing between network segments
- iptables chain order and rule precedence (top-down evaluation)
- Network troubleshooting methodology: verifying at each layer (routing table → firewall rules → packet capture) rather than guessing
- Principle of least privilege applied to network design
