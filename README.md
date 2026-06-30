# Home Lab: Network Segmentation with VLAN-style Isolation and Firewall Rules

## Overview

This project simulates a small business/home network with a dedicated guest network, isolated from the trusted internal network, using virtual machines and Linux-based routing and firewall rules. It demonstrates core networking and security concepts: subnetting, IP forwarding, routing, and stateful firewall policy — the same principles used in real corporate guest WiFi and VLAN deployments.

**Why this project**: As an Application Support Analyst, I work with production systems and incidents daily, but wanted hands-on experience with the infrastructure layer underneath — specifically how network segmentation and firewall policy are actually implemented, not just diagrammed.

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
- **tcpdump** — packet-level traffic verification

## What Was Built

### 1. Network Segmentation
Created two isolated subnets (`192.168.10.0/24` and `192.168.20.0/24`) using VirtualBox Internal Networks, simulating a trusted LAN and a guest LAN as they would exist on separate VLANs in a real switch-based network.

### 2. Routing
Enabled IP forwarding on `router-fw` (`net.ipv4.ip_forward=1`) so it could pass traffic between the two subnets and out to the internet via NAT, acting as the gateway for both networks.

### 3. Firewall Policy
Implemented isolation between guest and trusted networks using `iptables`:

```bash
# Allow replies to connections trusted-lan initiated
sudo iptables -I FORWARD -i enp0s9 -o enp0s8 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block new connections from guest-lan to trusted-lan
sudo iptables -I FORWARD -i enp0s9 -o enp0s8 -j DROP
```

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

Documenting real issues hit during the build — this is often more valuable to show than a clean success story, since it demonstrates actual troubleshooting ability.

1. **YAML indentation error in Netplan config** — `addresses:` was indented at the wrong level relative to its parent key, causing a "expected mapping, check indentation" error. Fixed by aligning it as a sibling of `dhcp4:` rather than `enp0sX:`.
2. **Duplicate static IP across two interfaces** — both `enp0s8` and `enp0s9` were accidentally assigned `192.168.10.1`, causing inconsistent `ip a` output. Diagnosed by inspecting the raw Netplan file rather than assuming the configuration was correct.
3. **Asymmetric routing misunderstanding** — initially assumed guest→trusted connectivity was working based on ping behavior, but `tcpdump` revealed packets weren't arriving at all. Root cause: neither client VM had a static route to the other's subnet — each only knew its own directly-connected network. This was a routing gap, not a firewall issue, and was diagnosed by checking `ip route` on both ends rather than assuming the firewall was the only variable.
4. **Firewall rule blocking legitimate return traffic** — an initial single-direction DROP rule blocked *all* traffic in that direction, including replies to connections the trusted side initiated. Fixed by adding a stateful `ESTABLISHED,RELATED` ACCEPT rule, ordered above the DROP rule, so iptables' top-down rule evaluation correctly allows replies while still blocking new guest-initiated connections.

## Key Concepts Demonstrated

- Subnetting and IP addressing
- IP forwarding / routing between network segments
- Stateful firewall rules (`ESTABLISHED`, `RELATED`) vs. simple directional blocking
- iptables chain order and rule precedence (top-down evaluation)
- Network troubleshooting methodology: verifying at each layer (routing table → firewall rules → packet capture) rather than guessing
- Principle of least privilege applied to network design

## Possible Extensions

- Add a second firewall rule blocking trusted→guest as well, for full bidirectional isolation
- Replace static `iptables` rules with `ufw` or a dedicated firewall VM (e.g. pfSense/OPNsense)
- Add a SIEM (e.g. Wazuh) to monitor and alert on blocked connection attempts
- Introduce a vulnerable VM on guest-lan and use Kali Linux to demonstrate why the isolation matters in practice
