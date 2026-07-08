# Cybersecurity Home Lab Setup

**Platform:** VirtualBox 7.2  
**Host OS:** Windows 11  
**Date:** May 2026  

---

## Objective

Build an isolated virtual lab environment to safely practice cybersecurity attacks and defenses without risk to the host machine or home network.

---

## Virtual Machines

| VM | OS | Role | IP Address |
|----|----|------|------------|
| Kali Linux | Kali 2026.1 | Attacker | 192.168.20.11 |
| Ubuntu | Ubuntu 26.04 | Victim/Defender | 192.168.20.10 |

---

## Network Configuration

### Why not NAT?

By default, VirtualBox assigns **NAT** to every VM. NAT gives the VM internet access through the host machine, useful for downloading tools, but dangerous when practicing attacks or running malware, as there is a risk of affecting the host network.

### Why Internal Network?

**Internal Network** isolates the VMs in their own private network. They can communicate with each other but:
- Cannot access the internet
- Cannot access the host machine
- Cannot reach the home LAN

This makes it the safest option for attack/defense practice.

### Network Diagram 

┌─────────────────────────────────────┐
│         Internal Network            │
│              "labnet"               │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │  Kali Linux  │  │   Ubuntu    │  │
│  │ 192.168.20.11│◄─►│192.168.20.10│  │
│  │  (Attacker)  │  │  (Victim)   │  │
│  └──────────────┘  └─────────────┘  │
│                                     │
│      ✗ No internet access           │
│      ✗ No access to host machine    │
└─────────────────────────────────────┘

---

## Static IP Configuration

### Kali Linux
Edited `/etc/network/interfaces`: 

auto eth0
iface eth0 inet static
address 192.168.20.11
netmask 255.255.255.0 

Applied with: sudo systemctl restart networking 

### Ubuntu
Configured via **Settings → Network → IPv4 → Manual**:
- Address: 192.168.20.10
- Netmask: 255.255.255.0
- Gateway: (empty)

---

## Connectivity Test

Verified communication between VMs using ping from Kali to Ubuntu: 
ping 192.168.20.10 
Result: 

64 bytes from 192.168.20.10: icmp_seq=1 ttl=64 time=2.21 ms
64 bytes from 192.168.20.10: icmp_seq=2 ttl=64 time=1.96 ms
64 bytes from 192.168.20.10: icmp_seq=3 ttl=64 time=1.61 ms 

✅ Both machines are communicating successfully on the isolated internal network.

---

## Key Takeaways

- VirtualBox network modes serve different purposes, choosing the right one is a security decision, not just a technical one
- Static IPs are essential when machines need to reliably find each other on a network
- Isolating lab environments from production networks is a fundamental security practice used in enterprise SOC environments

## Wazuh SIEM Deployment

### Overview
Deployed Wazuh 4.7.5 (all-in-one: Manager, Indexer, Dashboard) on Ubuntu to simulate 
a real SOC monitoring environment.

### Installation
```bash
curl -O https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
Dashboard accessible at: `https://127.0.0.1`

---

## Ubuntu Static IP via Netplan

Previous Ubuntu IP was configured via GUI. Reconfigured via netplan for server best practice.

Edited `/etc/netplan/01-network-manager-all.yaml`:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.20.10/24
```

Applied with:
```bash
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
sudo netplan apply
```

Verified with `ip addr show` — `inet 192.168.20.10/24` confirmed. ✅

---

## Wazuh Agent Deployment (Kali Linux)

### Installation
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor | sudo tee /usr/share/keyrings/wazuh.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install wazuh-agent=4.7.5-1 -y
```

### Configuration
Edited `/var/ossec/etc/ossec.conf` — set manager address:
```xml
<address>192.168.20.10</address>
```

### Registration
```bash
sudo /var/ossec/bin/agent-auth -m 192.168.20.10
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

Result: `INFO: Valid key received` ✅

---

## Troubleshooting & Lessons Learned

| Issue | Cause | Solution |
|-------|-------|----------|
| `netplan apply` failed | `systemd-networkd` not running | `sudo systemctl enable/start systemd-networkd` |
| Agent version mismatch | Kali installed 4.14.5, Manager is 4.7.5 | Reinstalled agent with `wazuh-agent=4.7.5-1` |
| `MANAGER_IP` not replaced | Agent installed without labnet access | Manually edited `ossec.conf` after install |
| Duplicate agent name | Old agent entry not fully removed | Used `manage_agents -r 002` + manager restart |
| Kali no internet on labnet | Internal Network has no internet by design | Temporarily switched to NAT for downloads |

**Key lesson:** Plan the installation order before starting. Install the agent while still on NAT, then switch to labnet and configure the static IP.

---

## Final Result

Wazuh dashboard showing:
- **Total agents: 1**
- **Active agents: 1** ✅

Kali Linux (attacker) successfully monitored by Wazuh SIEM on Ubuntu (defender).
