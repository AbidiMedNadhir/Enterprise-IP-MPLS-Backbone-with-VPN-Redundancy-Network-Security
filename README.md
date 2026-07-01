# 🌐 Enterprise IP/MPLS Backbone with VPN, Redundancy & Network Security

<div align="center">

![TEK-UP University](https://img.shields.io/badge/TEK--UP_University-SSIR--4--A-0046A0?style=for-the-badge)
![GNS3](https://img.shields.io/badge/Simulator-GNS3-FF6600?style=for-the-badge&logo=cisco)
![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco)
![pfSense](https://img.shields.io/badge/Firewall-pfSense_2.7.0-212121?style=for-the-badge)
![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix-D40000?style=for-the-badge)
![FreeRADIUS](https://img.shields.io/badge/AAA-FreeRADIUS-003D6B?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_Server-E95420?style=for-the-badge&logo=ubuntu)

*Report on the Design and Deployment of a Secure Network Infrastructure Based on an IP/MPLS Backbone*

**TEK-UP University — SSIR-4-A — Academic Year 2025/2026**
Supervised by **Mr. Tarek HDIJI**

</div>

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Chapter 1 — IP/MPLS Backbone & VPN](#-chapter-1--ipmpls-backbone--vpn)
- [Chapter 2 — Redundant Extended LAN](#-chapter-2--redundant-extended-lan)
- [Chapter 3 — Monitoring, AAA & pfSense Security](#-chapter-3--monitoring-aaa--pfsense-security)
- [Complete IP Addressing Plan](#-complete-ip-addressing-plan)
- [Technologies & Protocols](#-technologies--protocols)
- [Key Achievements](#-key-achievements)
- [References](#-references)

---

## 🔭 Project Overview

This project covers the full design and deployment of a **secure enterprise network infrastructure** structured across four complementary layers:

```
┌──────────────────────────────────────────────────────────────┐
│                     SECURITY LAYER                           │
│        pfSense Firewall (WAN / LAN / DMZ)                    │
│        FreeRADIUS — AAA Authentication & Authorization       │
├──────────────────────────────────────────────────────────────┤
│                   MONITORING LAYER                           │
│        Zabbix Server — SNMPv3 — Syslog — NTP — IP SLA       │
├──────────────────────────────────────────────────────────────┤
│                 EXTENDED LAN LAYER                           │
│        VLAN Segmentation — HSRP — EtherChannel — DHCP       │
├──────────────────────────────────────────────────────────────┤
│                   BACKBONE LAYER                             │
│        IP/MPLS — VRF CYBER — OSPF — MP-BGP — LDP            │
└──────────────────────────────────────────────────────────────┘
```

### Network Sites

| Site | Role | CE Router | Loopback |
|------|------|-----------|----------|
| **Headquarters 1** | Primary site | CE11 | `172.16.11.11/32` |
| **Headquarters 2** | Secondary site | CE21 | `172.16.21.21/32` |
| **Branch 1** | Remote site | CE12 | `172.16.12.12/32` |
| **Branch 2** | Remote site | CE22 | `172.16.22.22/32` |

---

## 📦 Chapter 1 — IP/MPLS Backbone & VPN

### Objective

Design and deployment of a fully redundant **IP/MPLS Backbone** with VPN isolation using a dedicated VRF instance per client.

### Network Roles

| Role | Devices | Function |
|------|---------|----------|
| **Provider (P)** | P1, P2 | Core label switching |
| **Provider Edge (PE)** | PE1, PE2 | VRF management, MP-BGP peering |
| **Customer Edge (CE)** | CE11, CE12, CE21, CE22 | Client-side OSPF routing |

### Protocols

| Protocol | Scope | Purpose |
|----------|-------|---------|
| **OSPF** | All routers | Dynamic routing — Area 0 (backbone) + client areas |
| **LDP** | P and PE routers | MPLS label distribution |
| **MP-BGP** | PE1 ↔ PE2 | VPN route exchange between provider edges |
| **VRF CYBER** | PE1, PE2 | Client traffic isolation |

### VRF Configuration

```cisco
ip vrf VPN_CYBER
  rd 100:3
  route-target both 100:3

interface GigabitEthernet3/0
  ip vrf forwarding VPN_CYBER
  ip address 192.168.1.1 255.255.255.252
```

### OSPF Area Design

```
Area 0  ───  Backbone core  (PE1, PE2, P1, P2)
Area 11 ───  Headquarters 1 (CE11 + PE1 G3/0)
Area 11 ───  Headquarters 2 (CE21 + PE1 G4/0)  ← unified with Area 11
Area 12 ───  Branch 1       (CE12 + PE2 G3/0)
Area 22 ───  Branch 2       (CE22 + PE2 G4/0)
```

> **Design decision:** Area 21 was replaced by Area 11 to unify the OSPF domain across both headquarters sites, ensuring consistent routing continuity with pfSense.

---

## 🔌 Chapter 2 — Redundant Extended LAN

### Objective

Deployment of a **three-layer hierarchical LAN** with gateway redundancy, VLAN segmentation, dynamic address allocation and full OSPF integration.

### Three-Layer Hierarchy

```
┌─────────────────────────────────────┐
│  CORE         →  IP/MPLS Backbone   │
├─────────────────────────────────────┤
│  DISTRIBUTION →  Fédérateur1        │
│               →  Fédérateur2        │  HSRP active/standby + EtherChannel trunk
├─────────────────────────────────────┤
│  ACCESS       →  SW1, SW2           │  Headquarters
│               →  SW3, SW4           │  Branch sites
└─────────────────────────────────────┘
```

### VLAN Plan

#### Headquarters

| VLAN | Name | Network |
|------|------|---------|
| VLAN 10 | TEKUP1 | `172.16.210.0/24` |
| VLAN 15 | TEKUP2 | `172.16.215.0/24` |
| VLAN 20 | Management | `172.16.220.0/24` |
| VLAN 300 | WAN\_OSPF | `192.168.1.44/30` |

#### Branch 1 (CE12)

| VLAN | Name | Network |
|------|------|---------|
| VLAN 201 | Management | `172.16.201.0/24` |
| VLAN 202 | DATA | `172.16.202.0/24` |

#### Branch 2 (CE22)

| VLAN | Name | Network |
|------|------|---------|
| VLAN 203 | Management | `172.16.203.0/24` |
| VLAN 204 | DATA | `172.16.204.0/24` |

### Gateway Redundancy — HSRP

```
VLANs 10 / 15 / 20 :
  ├── Virtual IP   →  Default gateway for all hosts
  ├── Fédérateur1  →  Active router   (higher priority)
  └── Fédérateur2  →  Standby router  (automatic failover)
```

### Additional Services

| Service | Details |
|---------|---------|
| **EtherChannel** | TRUNK mode, VLANs 10/15/20/300 allowed |
| **DHCP** | Fédérateur1 & 2 for VLAN 10/15 — CE12/CE22 for branch VLANs |
| **OSPF** | `passive-interface default` on Fédérateurs — VLAN 300 active (cost 3000) |
| **Trunk links** | Native VLAN 20 on all inter-switch links |

---

## 🔒 Chapter 3 — Monitoring, AAA & pfSense Security

### 3.1 — Centralized Monitoring with Zabbix

**Zabbix Server address:** `172.16.220.250/24` — Ubuntu Server on Management VLAN 20

The Zabbix platform provides real-time supervision of all network devices through SNMPv3, centralized log collection via Syslog, time synchronization via NTP, and end-to-end performance measurement via IP SLA probes deployed on all CE routers.

#### SNMPv3 Configuration (applied to all devices)

```cisco
snmp-server group TEKUP v3 auth
snmp-server user TEKUP TEKUP v3 auth md5 SSIR
snmp-server host 172.16.220.250 version 3 auth TEKUP
snmp-server enable traps snmp linkdown linkup
snmp-server enable traps ospf
snmp-server enable traps config
```

#### Syslog & NTP

```cisco
logging on
logging 172.16.220.250
logging trap warning

ntp server 172.16.220.250
```

#### IP SLA — Branch to Headquarters Probes

```cisco
! On Branch routers (CE12, CE22)
ip sla 11
  udp-jitter 192.168.1.2 frequency 300
ip sla schedule 11 start-time now life forever

! On Headquarters routers (CE11, CE21)
ip sla responder
```

---

### 3.2 — AAA Authentication with FreeRADIUS

**FreeRADIUS Server address:** `172.16.220.200/24` — Ubuntu Server on Management VLAN 20

#### Authentication Methods

| Priority | Method | Description |
|----------|--------|-------------|
| **Method 1** | RADIUS Server `172.16.220.200` | Centralized authentication |
| **Method 2** | Local fallback | Backup in case of server unavailability |

#### Privilege Levels

| Level | Access Type |
|-------|------------|
| Privilege 1 | Read-only access |
| Privilege 10 | Intermediate operator access |
| Privilege 15 | Full administrative access |

#### Device Configuration

```cisco
aaa new-model
radius-server host 172.16.220.200 key VPN_Cyber
aaa authentication login default group radius local
aaa authorization exec default group radius local
username admin privilege 15 secret <password>
```

---

### 3.3 — Perimeter Security with pfSense

**pfSense 2.7.0** deployed as a single perimeter firewall separating three distinct security zones.

#### Zone Architecture

```
                    ┌────────────────────────┐
                    │        pfSense          │
                    │     2.7.0 Firewall      │
          ┌─────────┤  OPT1 │  LAN │  OPT2  ├──────────┐
          │         └───────┴──────┴─────────┘          │
          │                  │                           │
    CE11 G1/0          Fédérateur1                  Zone DMZ
  192.168.1.2/30    172.16.220.0/24            172.16.255.0/25
  (MPLS Backbone)  (Management Network)        (Exposed Servers)
```

#### Interface Assignment

| Interface | Connected To | IP Address |
|-----------|-------------|------------|
| **OPT1 (WAN)** | CE11 router | `192.168.1.1/30` |
| **LAN** | Fédérateur1 | `172.16.220.1/24` |
| **OPT2 (DMZ)** | DMZ servers | `172.16.255.1/25` |

#### DMZ — Exposed Services

| Server | Service | Network | Gateway |
|--------|---------|---------|---------|
| SMTP Server | Email | `172.16.255.0/25` | `172.16.255.1` |
| DNS Server | Name resolution | `172.16.255.0/25` | `172.16.255.1` |
| WWW Server | Web | `172.16.255.0/25` | `172.16.255.1` |

#### Firewall Policy

```
✅ ALLOWED  :  HQ (VLAN10/15) + Branches (VLAN202/204)  →  DMZ (SMTP / DNS / WWW)
✅ ALLOWED  :  Admin1 (172.16.220.100) + Admin2 (172.16.220.150)  →  DMZ (HTTP/HTTPS/SSH/Telnet/ICMP)
❌ BLOCKED  :  HQ LAN  ↔  Branch LAN  (direct inter-site communication)
```

#### OSPF Integration on pfSense

```
Area 11 — configured on all three interfaces:
  ├── LAN   →  172.16.220.0/24   (toward Fédérateur1)
  ├── OPT1  →  192.168.1.0/30    (toward CE11)
  └── OPT2  →  172.16.255.0/25   (toward DMZ)
```

---

## 📊 Complete IP Addressing Plan

### MPLS Backbone Links

| Link | Network | Device A | Device B |
|------|---------|----------|----------|
| PE1 – P1 | `10.1.1.0/30` | PE1 G1/0 = `.1` | P1 G2/0 = `.2` |
| PE1 – P2 | `10.1.1.4/30` | PE1 G2/0 = `.5` | P2 G3/0 = `.6` |
| PE2 – P2 | `10.1.1.8/30` | PE2 G1/0 = `.9` | P2 G2/0 = `.10` |
| PE2 – P1 | `10.1.1.12/30` | PE2 G2/0 = `.13` | P1 G3/0 = `.14` |
| P1 – P2 | `10.1.1.20/30` | P1 G1/0 = `.21` | P2 G1/0 = `.22` |

### Loopback Addresses

| Router | Loopback0 |
|--------|-----------|
| PE1 | `1.1.1.1/32` |
| PE2 | `2.2.2.2/32` |
| P1 | `3.3.3.3/32` |
| P2 | `4.4.4.4/32` |
| CE11 | `172.16.11.11/32` |
| CE12 | `172.16.12.12/32` |
| CE21 | `172.16.21.21/32` |
| CE22 | `172.16.22.22/32` |

### WAN Links — CE to PE (VRF CYBER)

| Link | Network | CE G1/0 | PE Interface |
|------|---------|---------|--------------|
| CE11 – PE1 | `192.168.1.0/30` | `.2` | PE1 G3/0 = `.1` |
| CE21 – PE1 | `192.168.1.4/30` | `.6` | PE1 G4/0 = `.5` |
| CE12 – PE2 | `192.168.1.8/30` | `.10` | PE2 G3/0 = `.9` |
| CE22 – PE2 | `192.168.1.12/30` | `.14` | PE2 G4/0 = `.13` |

> ⚠️ The entire `192.168.1.0/24` range is **exclusively reserved** for CE-to-PE WAN links on the MPLS backbone.

### Management Network (VLAN 20 — `172.16.220.0/24`)

| Device | IP Address |
|--------|-----------|
| pfSense LAN | `172.16.220.1` |
| Fédérateur1 | `172.16.220.2` |
| Fédérateur2 | `172.16.220.3` |
| Admin1 | `172.16.220.100` |
| SW1 | `172.16.220.101` |
| SW2 | `172.16.220.102` |
| Admin2 | `172.16.220.150` |
| FreeRADIUS (AAA) | `172.16.220.200` |
| Zabbix (Monitoring) | `172.16.220.250` |

### DMZ Network (`172.16.255.0/25`)

| Device | Role | Gateway |
|--------|------|---------|
| SMTP / DNS / WWW Servers | Exposed services | `172.16.255.1` (pfSense OPT2) |

---

## 🛠️ Technologies & Protocols

### Simulation Environment

| Tool | Purpose |
|------|---------|
| **GNS3** | Cisco router and switch emulation |
| **GNS3 VM** | Performance optimization for heavy topologies |
| **VMware Workstation** | Hosting of Linux servers (Zabbix, FreeRADIUS, pfSense) |

### Network Devices

| Type | Model | Instances |
|------|-------|-----------|
| Routers | Cisco C700 | CE11, CE12, CE21, CE22, PE1, PE2, P1, P2 |
| Layer 3 Switch | Cisco 3650-24PS | Fédérateur1, Fédérateur2 |
| Layer 2 Switch | Cisco 2960-24TT | SW1, SW2, SW3, SW4 |

### Software Stack

| Software | Version | Role |
|----------|---------|------|
| **pfSense** | 2.7.0 | Perimeter firewall — WAN/LAN/DMZ |
| **Zabbix** | LTS | Centralized network monitoring |
| **FreeRADIUS** | 3.0 | AAA server — RADIUS authentication |
| **MySQL** | — | Zabbix backend database |
| **Ubuntu Server** | LTS | Host OS for all servers |
| **Apache + PHP** | — | Zabbix web frontend |

### Protocol Stack

| Protocol | Layer | Role |
|----------|-------|------|
| **OSPF** | Network | Dynamic routing — backbone + client sites |
| **MP-BGP** | Application | VPN route distribution between PE routers |
| **LDP** | Network | MPLS label distribution |
| **VRF** | Network | Per-client traffic isolation (VRF CYBER) |
| **HSRP** | Network | Gateway redundancy on distribution layer |
| **EtherChannel** | Data Link | Link aggregation between Fédérateurs |
| **SNMPv3** | Application | Authenticated network supervision |
| **RADIUS** | Application | Centralized AAA |
| **SSH v2** | Application | Secure device management |
| **NTP** | Application | Centralized time synchronization |
| **Syslog** | Application | Centralized log collection |
| **IP SLA** | Application | Latency, jitter and packet loss measurement |

---

## ✅ Key Achievements

- Full **IP/MPLS Backbone** with double PE-to-P redundancy on every provider edge
- **VRF CYBER** deployed on PE1 and PE2 ensuring strict client traffic isolation via MP-BGP
- **Hierarchical LAN** with HSRP gateway redundancy, EtherChannel aggregation and per-VLAN DHCP across 4 sites
- **Zabbix** monitoring platform with SNMPv3, Syslog and NTP covering all 10 network devices
- **FreeRADIUS** AAA with 3 privilege levels, RADIUS primary and local fallback authentication
- **pfSense** firewall enforcing WAN/LAN/DMZ zone separation with stateful packet filtering
- **OSPF Area 11** unified across pfSense, CE11 and both Fédérateurs for seamless route propagation
- **Security policy** enforced: DMZ access authorized from all sites, direct inter-LAN traffic blocked

---

## 📚 References

- RFC 3031 — Multiprotocol Label Switching Architecture
- RFC 4364 — BGP/MPLS IP Virtual Private Networks
- RFC 2328 — OSPF Version 2
- RFC 2865 — Remote Authentication Dial In User Service (RADIUS)
- [pfSense Official Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Zabbix Official Documentation](https://www.zabbix.com/documentation/current/)
- [FreeRADIUS Official Documentation](https://freeradius.org/documentation/)

---

<div align="center">

**TEK-UP University — SSIR-4-A — 2025/2026**
Supervised by **Mr. Tarek HDIJI**

![GitHub](https://img.shields.io/badge/Report-PDF_Available-181717?style=flat-square&logo=github)
![License](https://img.shields.io/badge/License-Academic_Use_Only-blue?style=flat-square)

</div>
