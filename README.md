# Home Lab: Proxmox VE SOC Range & Wazuh SIEM Pipeline

**Author:** Maalik Barnes  
**Certifications:** CompTIA Security+ | CompTIA A+ | CompTIA Network+ (In Progress)  
**Environment:** Proxmox VE 8.x • OPNsense • Wazuh SIEM • OpenSearch • Debian 12 LXC • `iptables`

*(Note: For security, internal management IP addresses and hardware identifiers in this documentation have been sanitized to generic ranges.)*

---

## Overview

This repository documents the architecture, deployment, and troubleshooting of a bare-metal Security Operations Center home lab built on a Dell OptiPlex 7000 series server. 

The goal of this build was to move past pre-built virtual machine exercises and engineer a segmented network environment from the hardware layer up. The lab includes Layer 2/Layer 3 network isolation via virtual switching, stateful firewall routing with OPNsense, persistent Linux syslog forwarding, active File Integrity Monitoring, attack simulation mapped to MITRE ATT&CK, and automated host-level IP blocking using Wazuh.

---

## Network Architecture & Topology

To test detections safely without exposing the primary LAN (`192.168.1.0/24`), the hypervisor uses a dual-bridge layout. `vmbr0` bridges to the physical network interface (`nic0`) for management and SIEM dashboard access, while `vmbr1` acts as an isolated virtual switch in system RAM with no physical uplink. An OPNsense firewall sits between the two bridges to control traffic between the target subnet (`10.0.10.0/24`) and the upstream network.

```mermaid
flowchart TB
    subgraph HomeLAN["Physical Management LAN (192.168.1.0/24)"]
        Router["Primary Edge Gateway\n192.168.1.1"]
        Switch["Network Switch"]
        Workstation["Management Workstation\n(SSH Ed25519 / HTTPS)"]
        Router <--> Switch
    end

    subgraph ProxmoxHost["Bare-Metal Hypervisor — Proxmox VE (24GB DDR4 RAM)"]
        NIC0["Physical NIC (nic0)"]
        
        subgraph VMBR0["vmbr0 — Management & SIEM Bridge"]
            PVE["Proxmox Host (pve)\n192.168.1.200 (:8006 / :22)"]
            Wazuh["CT 100: Wazuh SIEM & OpenSearch\n192.168.1.235 (:443 / :1514 / :1515)"]
        end

        subgraph Firewall["VM 101: OPNsense Firewall"]
            WAN["vtnet0 (WAN)\n192.168.1.78"]
            LAN["vtnet1 (LAN Gateway)\n10.0.10.1/24"]
            Rules["LAN Firewall Rules (Top-to-Bottom):\n1. PASS -> 192.168.1.235 (UDP 1514 Telemetry)\n2. BLOCK -> 192.168.1.0/24 (RFC1918 LAN Isolation)\n3. PASS -> * (Outbound NAT Updates)"]
            WAN --- Rules --- LAN
        end

        subgraph VMBR1["vmbr1 — Isolated Target Bridge (10.0.10.0/24 — No Physical Ports)"]
            Target["CT 102: Audit-Target-01 (Debian 12)\n10.0.10.138\n[Wazuh Agent • rsyslog • syscheck FIM • iptables]"]
        end
    end

    Switch <-->|Ethernet| Workstation
    Switch <-->|Ethernet| NIC0
    NIC0 <--> VMBR0
    VMBR0 <--> WAN
    LAN <--> VMBR1
    Target -.->|Encrypted Logs (UDP 1514)| Wazuh
```

---

## System Specifications

| Component | Details | Role in Lab |
| :--- | :--- | :--- |
| **Physical Host** | Dell OptiPlex 7000-Series SFF | Bare-metal server; 24GB Non-ECC DDR4 RAM; SSD (`ext4`  which was selected over ZFS to conserve the host RAM) |
| **Hypervisor** | Proxmox VE 8.x | Host OS (`192.168.1.200`); manages KVM virtual machines, LXC containers, and virtual bridges |
| **Firewall Gateway** | OPNsense (VM 101) | Stateful packet filtering, outbound NAT, DHCP server (`10.0.10.1`), and subnet isolation |
| **SIEM Stack** | Wazuh + OpenSearch (CT 100) | Privileged Debian 12 LXC (`192.168.1.235`); log collector, rule correlation engine, and web UI |
| **Target Endpoint** | `Audit-Target-01` (CT 102) | Unprivileged Debian 12 LXC (`10.0.10.138`); runs Wazuh Agent, `rsyslog`, `syscheck`, and `iptables` |

---

## Implementation Breakdown

### 1. Hardware Upgrade & Hypervisor Hardening
* **Memory Upgrade**: Scaled host memory from 8GB to 24GB DDR4 RAM to handle OpenSearch indexing alongside virtual machines.
* **SSH & Firewall Hardening**: Configured Ed25519 SSH key authentication, set `PermitRootLogin prohibit-password` in `/etc/ssh/sshd_config`, and then enabled the Proxmox host firewall restricting ports `8006` and `22` to the local LAN subnet. *(Note: `prohibit-password` is used rather than `no` to preserve compatibility with Proxmox VE's native `pmxcfs` cluster/container management subsystems while completely removing the need for password authentication.)*
* **Package Repositories & DNS**: Disabled Proxmox's default commercial subscription repositories (which returned `HTTP 401 Unauthorized` errors during updates) and configured the `pve-no-subscription` community repository. Configured Quad9 (`9.9.9.9`) for DNS resolution and updated local root certificates (`ca-certificates`).

### 2. Network Segmentation & Firewall Rules
* **Virtual Bridge Setup**: Created `vmbr1` in Proxmox with no physical ports attached so traffic from target containers stays contained in the memory unless routed through OPNsense.
* **Fixing the OPNsense Default-Allow Rule**: By default, OPNsense includes an "Allow LAN to any" rule on its LAN interface. During initial connectivity testing from `Audit-Target-01` (`10.0.10.138`), I noticed the target container could ping devices on the `192.168.1.0/24` management network. To close this gap while keeping SIEM logging functional, I restructured the `LAN` interface rules in hierarchal order:

| Priority | Action | Source | Destination | Port / Protocol | Purpose |
| :---: | :---: | :--- | :--- | :--- | :--- |
| **1** | `PASS` | `LAN net (10.0.10.0/24)` | `192.168.1.235` (Wazuh) | `UDP 1514` | Allows encrypted Wazuh agent log forwarding across the firewall |
| **2** | `BLOCK` | `LAN net (10.0.10.0/24)` | `192.168.1.0/24` (Primary LAN) | `*` (Any) | Blocks traffic from the labs subnet to the primary network |
| **3** | `PASS` | `LAN net (10.0.10.0/24)` | `*` (External WAN) | `*` (Any) | Allows outbound internet traffic via NAT for package updates |

### 3. SIEM Deployment & Log Pipeline Configuration
* **Host Kernel Tuning**: Increased the host virtual memory map limit in `/etc/sysctl.conf` and deployed the Wazuh stack in a privileged LXC container so OpenSearch could initialize without kernel permission errors.
* **Enabling Persistent Syslog (`rsyslog`)**: Minimal Debian 12 LXC templates use `systemd-journald` in volatile memory and do not create `/var/log/auth.log` by default. Because the Wazuh agent checks flat log files for SSH events, I installed and enabled `rsyslog` on `Audit-Target-01` to write any authentication events to the disk.
* **Baseline False Positive Review**: Investigated initial Leve 7 `rootcheck` alerts (`File '/dev/.lxc/proc/interrupts' present on /dev. Possible hidden file.`) and verified they were normal Proxmox LXC virtualization mounts rather than malicious artifacts.

### 4. Real Time File Integrity Monitoring 
* Updated the `<syscheck>` block in `/var/ossec/etc/ossec.conf` on `Audit-Target-01` to enable real-time kernel `inotify` monitoring and line-by-line diff tracking (`report_changes="yes"`) on core identity and configuration files:

```xml
<directories check_all="yes" realtime="yes" report_changes="yes">/etc/passwd,/etc/shadow,/etc/ssh/sshd_config</directories>
```

* **Verification**: Appended a test comment (`echo "# SOC_LAB_FIM_TEST" >> /etc/passwd`) and verified that Wazuh immediately generated a **Rule 550** (`Integrity checksum changed`) alert displaying the old and new SHA-256 hashes alongside the exact text diff, then reverted the file (`sed -i '/# SOC_LAB_FIM_TEST/d' /etc/passwd`).

### 5. Attack Simulation & Detection Analysis
Ran controlled tests on `Audit-Target-01` (`10.0.10.138`) to verify log decoding, severity scoring, and framework mappings (`MITRE ATT&CK`, `NIST 800-53`, `PCI DSS`):

#### Test 1: Local Account Persistence (`MITRE ATT&CK T1136 - Create Account`)
* **Command Executed**: `useradd -m -s /bin/bash adversary_account`
* **Captured Alert JSON (`Rule 5902` — Level 8)**:

```json
{
  "agent": { "id": "001", "name": "Audit-Target-01", "ip": "10.0.10.138" },
  "data": {
    "uid": "1000",
    "gid": "1000",
    "dstuser": "adversary_account",
    "home": "/home/adversary_account",
    "shell": "/bin/bash"
  },
  "rule": {
    "id": "5902",
    "level": 8,
    "description": "New user added to the system.",
    "mitre": {
      "id": [ "T1136" ],
      "tactic": [ "Persistence" ],
      "technique": [ "Create Account" ]
    },
    "nist_800_53": [ "AU.14", "AC.7", "AC.2", "IA.4" ],
    "pci_dss": [ "10.2.7", "10.2.5", "8.1.2" ]
  },
  "full_log": "Sep 05 22:25:49 Audit-Target-01 useradd[4886]: new user: name=adversary_account, UID=1000, GID=1000, home=/home/adversary_account, shell=/bin/bash, from=/dev/pts/3"
}
```

#### Test 2: SSH Brute-Force (`MITRE ATT&CK T1110 - Brute Force`)
* **Simulation**: Generated 8 rapid failed root SSH login events from RFC 5737 documentation test IP `198.51.100.23`:
```bash
for i in {1..8}; do
  logger -p auth.warn -t sshd "Failed password for root from 198.51.100.23 port 41234 ssh2"
done
```
* **Detection**: Escalated from individual authentication failure events to **Wazuh Rule 5763** (*"sshd: brute force trying to get access to the system. Authentication failed."*, **Level 10**, MITRE `T1110`, NIST 800-53 `SI.4`, `AU.14`, `AC.7`).

### 6. Automated Containment (Wazuh Active Response)
Configured an Active Response block in `/var/ossec/etc/ossec.conf` on the Wazuh Manager to trigger the `firewall-drop` script on the target endpoint whenever authentication failure rules fire:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_group>authentication_failures</rules_group>
  <timeout>60</timeout>
</active-response>
```

* **Containment Verification (`/var/ossec/logs/active-responses.log`)**:
  * **`23:31:30`**: 8 failed root SSH attempts logged from `198.51.100.23`.
  * **`23:31:32` (`command: add`)**: Wazuh tripped **Rule 5763** (Level 10) and instructed `wazuh-execd` on `Audit-Target-01` to insert an `iptables` INPUT drop rule for `198.51.100.23` within 2 seconds.
  * **`23:32:33` (`command: delete`)**: After the 60-second timeout expired, `firewall-drop` removed the rule from `iptables` automatically to prevent permanent lockouts.

```json
{
  "version": 1,
  "origin": { "name": "node01", "module": "wazuh-execd" },
  "command": "add",
  "parameters": {
    "alert": {
      "rule": {
        "id": "5763",
        "level": 10,
        "description": "sshd: brute force trying to get access to the system. Authentication failed.",
        "mitre": { "id": ["T1110"], "tactic": ["Credential Access"], "technique": ["Brute Force"] }
      },
      "agent": { "id": "001", "name": "Audit-Target-01", "ip": "10.0.10.138" },
      "data": { "srcip": "198.51.100.23", "srcport": "41234", "dstuser": "root" }
    },
    "program": "active-response/bin/firewall-drop"
  }
}
```
```json
{
  "version": 1,
  "origin": { "name": "node01", "module": "wazuh-execd" },
  "command": "delete",
  "parameters": {
    "alert": {
      "rule": {
        "id": "5763",
        "level": 10,
        "description": "sshd: brute force trying to get access to the system. Authentication failed."
      },
      "agent": { "id": "001", "name": "Audit-Target-01", "ip": "10.0.10.138" },
      "data": { "srcip": "198.51.100.23", "srcport": "41234", "dstuser": "root" }
    },
    "program": "active-response/bin/firewall-drop"
  }
}
```

---

## Troubleshooting & Incident Log

Below is a record of hardware, networking, and Linux configuration issues I resolved while building this environment:

| Issue / Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **Host Failed POST (1-3-2 Beep Code)** | Two installed RAM sticks were server-grade **ECC Unbuffered** memory (`KVR24E17S8/8MA`), which is incompatible with the desktop Intel Q170 chipset. | Pulled the CMOS battery to clear memory training cache, removed the ECC sticks, and installed 24GB of compatible non-ECC desktop DDR4 RAM. |
| **Post-Reboot SSH Lockout & Missing GUI Firewall Rules** | Changing the host IP to `192.168.1.200` without updating `/etc/hosts` prevented the Proxmox cluster filesystem (`pmxcfs`) from mounting on boot, putting `pve-firewall` into fail-shut mode. | Used the local web console to update `/etc/hosts` via a Bash script, and fixed a firewall rule macro that was set to `Ceph` instead of `SSH` (Port 22). |
| **Host DNS Failure (`Could not resolve host`)** | `/etc/resolv.conf` was pointing to loopback `127.0.0.1` with no local DNS caching service running. | Updated `/etc/resolv.conf` to use Quad9 (`9.9.9.9`) and installed `ca-certificates` so `curl` could verify GitHub SSL certificates. |
| **OpenSearch Container Crash on Startup** | Default host kernel memory map limits (`vm.max_map_count=65530`) and unprivileged LXC restrictions blocked OpenSearch from starting. | Set `vm.max_map_count=262144` on the Proxmox host and rebuilt the Wazuh container as a privileged LXC. |
| **Workstation Browser Timeout to Wazuh UI** | Active VPN client on the Windows workstation was intercepting and dropping local RFC 1918 subnet traffic. | Configured a split-tunnel exclusion rule in the VPN client to allow direct local LAN connections. |
| **OPNsense Swapped Interfaces & LAN Leak** | Initial interface assignment swapped WAN/LAN ports, and OPNsense's default-allow rule allowed `10.0.10.x` targets to ping `192.168.1.x` devices. | Reassigned `vtnet0` (WAN) and `vtnet1` (LAN), then added a block rule for `192.168.1.0/24` directly beneath the Wazuh UDP 1514 allow rule. |
| **Wazuh Agent Repo GPG Failure on Target** | Installing `wazuh-agent` on `Audit-Target-01` failed because the minimal Debian template did not include `gnupg` or `curl`. | Removed the broken repo file, installed `gnupg` and `curl`, imported the GPG key to `/usr/share/keyrings/wazuh.gpg`, and registered the agent. |
| **Missing SSH Logs on Target Container** | Minimal Debian 12 LXC templates only log to volatile `systemd-journald` buffers and do not create `/var/log/auth.log`. | Installed and enabled `rsyslog` on `Audit-Target-01` and restarted `wazuh-agent` to forward authentication events. |

---

## Lessons Learned & What I Would Do Differently

1. **Always Verify Firewall Assumptions with Live Traffic Tests**: Assuming a subnet is isolated just because it sits behind a firewall is dangerous. Running simple `ping` tests from `Audit-Target-01` before executing any attacks exposed OPNsense's default-allow LAN rule early and prevented lab traffic from touching the primary network.
2. **Privileged LXC vs. Dedicated KVM Virtual Machine for the SIEM**: Running Wazuh and OpenSearch inside a Privileged LXC container saved significant RAM overhead compared to a full VM, but privileged containers share the host kernel with reduced namespace isolation. In a production environment with greater hardware capacity, I would deploy the SIEM stack inside an isolated KVM Virtual Machine (or dedicated hardware) and place it on a dedicated Management/Security VLAN rather than the main LAN bridge.
3. **Minimal OS Templates Require Explicit Logging Audits**: Deploying lean container templates reduces attack surface, but often strips out standard utilities (`rsyslog`, `gnupg`, `curl`). Verifying that required log files actually exist on disk should always be step one before troubleshooting SIEM decoders.
4. **Future Lab Roadmap**: Next steps for this environment include deploying Linux `auditd` on the target node to capture kernel-level process execution (`execve`) and evaluating C2 beaconing detection across the OPNsense firewall boundary.

---

## Contact

* **LinkedIn:** [linkedin.com/in/maalikbarnes](https://linkedin.com/in/maalikbarnes)
* **Email:** [maalikbarnes@pm.me](mailto:maalikbarnes@pm.me)
