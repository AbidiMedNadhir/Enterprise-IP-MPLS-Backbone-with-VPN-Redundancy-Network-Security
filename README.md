# 🌐 Secure IP/MPLS Backbone Network — Architecture & Security Project

<div align="center">

![TEK-UP University](https://img.shields.io/badge/TEK--UP_University-SSIR--4--A-0046A0?style=for-the-badge&logo=graduation-cap)
![GNS3](https://img.shields.io/badge/Simulator-GNS3-FF6600?style=for-the-badge&logo=cisco)
![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco)
![pfSense](https://img.shields.io/badge/Firewall-pfSense_2.7.0-212121?style=for-the-badge)
![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix-D40000?style=for-the-badge)
![FreeRADIUS](https://img.shields.io/badge/AAA-FreeRADIUS-003D6B?style=for-the-badge)

**Rapport de Projet — Architecture et Sécurité de Réseau**  
Spécialité : Sécurité des Systèmes Informatiques et des Réseaux

</div>

---

## 👥 Équipe

| Nom | Rôle |
|-----|------|
| **Mohamed Nadhir Abidi** | Étudiant — SSIR-4-A |
| **Moadh Ben Naceur** | Étudiant — SSIR-4-A |
| **Mohamed Amine Rhimi** | Étudiant — SSIR-4-A |

> **Encadrant :** Mr. Tarek HDIJI — TEK-UP University

---

## 📋 Table des matières

- [Vue d'ensemble du projet](#-vue-densemble-du-projet)
- [Architecture générale](#-architecture-générale)
- [Chapitre 1 — Backbone VPN-MPLS](#-chapitre-1--backbone-vpn-mpls)
- [Chapitre 2 — LAN Étendu Redondant](#-chapitre-2--lan-étendu-redondant)
- [Chapitre 3 — Monitoring, AAA & Sécurité pfSense](#-chapitre-3--monitoring-aaa--sécurité-pfsense)
- [Plan d'adressage complet](#-plan-dadressage-complet)
- [Technologies utilisées](#-technologies-utilisées)
- [Structure du dépôt](#-structure-du-dépôt)
- [Comment compiler le rapport](#-comment-compiler-le-rapport)

---

## 🔭 Vue d'ensemble du projet

Ce projet académique conçoit et déploie une infrastructure réseau d'entreprise **complète et sécurisée**, couvrant trois couches complémentaires :

```
┌─────────────────────────────────────────────────────────┐
│              COUCHE SÉCURITÉ                            │
│   pfSense (WAN / LAN / DMZ)  +  FreeRADIUS (AAA)       │
├─────────────────────────────────────────────────────────┤
│              COUCHE SUPERVISION                         │
│   Zabbix Server  +  SNMPv3  +  Syslog  +  NTP          │
├─────────────────────────────────────────────────────────┤
│              COUCHE LAN ÉTENDU                          │
│   VLAN  +  HSRP  +  EtherChannel  +  DHCP  +  OSPF     │
├─────────────────────────────────────────────────────────┤
│              COUCHE BACKBONE                            │
│   IP/MPLS  +  VRF CYBER  +  OSPF  +  MP-BGP  +  LDP    │
└─────────────────────────────────────────────────────────┘
```

### Sites du réseau

| Identifiant | Rôle | Routeur CE |
|-------------|------|------------|
| **Siège 1** | Site principal | CE11 |
| **Siège 2** | Site principal secondaire | CE21 |
| **Branche 1** | Site distant | CE12 |
| **Branche 2** | Site distant | CE22 |

---

## 🏗️ Architecture générale

La topologie repose sur un **Backbone IP/MPLS** avec 2 routeurs Provider (P1, P2), 2 routeurs Provider Edge (PE1, PE2) et 4 routeurs Customer Edge (CE11, CE12, CE21, CE22).

```
        CE11 (Siège1)          CE21 (Siège2)
            │  G1/0                │  G1/0
            │  192.168.1.2         │  192.168.1.6
            │                      │
         G3/0│                  G4/0│
          ┌──┴──────────────────────┴──┐
          │           PE1              │
          │       (1.1.1.1/32)         │
          │  G1/0──────────── G2/0     │
          └──────┬───────────┬─────────┘
               P1│           │P2
          ┌──────┴──┐     ┌──┴──────┐
          │   P1    │─────│   P2    │
          │3.3.3.3  │     │4.4.4.4  │
          └──────┬──┘     └──┬──────┘
                 │            │
          ┌──────┴────────────┴──────────┐
          │           PE2                │
          │       (2.2.2.2/32)           │
          │  G3/0──────────── G4/0       │
          └──────┬───────────────┬───────┘
              G1/0│           G1/0│
          192.168.1.10      192.168.1.14
              │                   │
        CE12 (Branch1)      CE22 (Branch2)
```

---

## 📦 Chapitre 1 — Backbone VPN-MPLS

### Objectif

Déploiement d'un **Backbone IP/MPLS** avec VPN et isolation des trafics clients par VRF.

### Composants

| Rôle | Équipement | Description |
|------|-----------|-------------|
| **Provider (P)** | P1, P2 | Cœur du réseau — commutation des labels |
| **Provider Edge (PE)** | PE1, PE2 | Bordure MPLS — gestion VRF et MP-BGP |
| **Customer Edge (CE)** | CE11, CE12, CE21, CE22 | Routeurs clients — OSPF côté client |

### Protocoles implémentés

| Protocole | Rôle | Portée |
|-----------|------|--------|
| **OSPF** | Routage dynamique interne | Backbone (Area 0) + Sites (Area 11, 12, 21, 22) |
| **LDP** | Distribution des labels MPLS | Tous les routeurs P et PE |
| **MP-BGP** | Échange des routes VPN | Entre PE1 et PE2 |
| **VRF CYBER** | Isolation du trafic client | PE1 et PE2 — interfaces G3/0 et G4/0 |

### VRF configurée

```cisco
ip vrf VPN_CYBER
  rd 100:3
  route-target both 100:3
```

### Organisation OSPF

```
Area 0  ─── Backbone (PE1, PE2, P1, P2)
Area 11 ─── CE11 (Siège 1)   + PE1 G3/0
Area 21 ─── CE21 (Siège 2)   + PE1 G4/0  → remplacée par Area 11
Area 12 ─── CE12 (Branch 1)  + PE2 G3/0
Area 22 ─── CE22 (Branch 2)  + PE2 G4/0
```

> **Note :** L'Area 21 a été remplacée par l'Area 11 pour unifier le domaine OSPF du Siège.

---

## 🔌 Chapitre 2 — LAN Étendu Redondant

### Objectif

Déploiement d'un **LAN étendu hiérarchique** avec redondance HSRP, segmentation VLAN et distribution DHCP.

### Modèle hiérarchique

```
┌─────────────────────────────────┐
│  COUCHE CŒUR  →  Backbone MPLS  │
├─────────────────────────────────┤
│  DISTRIBUTION →  Fédérateur1    │
│               →  Fédérateur2    │  ← HSRP + EtherChannel
├─────────────────────────────────┤
│  ACCÈS        →  SW1, SW2       │  ← Siège
│               →  SW3, SW4       │  ← Branches
└─────────────────────────────────┘
```

### VLANs configurés

#### Siège principal

| VLAN | Nom | Réseau |
|------|-----|--------|
| VLAN 10 | TEKUP1 | `172.16.210.0/24` |
| VLAN 15 | TEKUP2 | `172.16.215.0/24` |
| VLAN 20 | Management | `172.16.220.0/24` |
| VLAN 300 | WAN_OSPF | `192.168.1.44/30` |

#### Branche 1 (CE12)

| VLAN | Nom | Réseau |
|------|-----|--------|
| VLAN 201 | Management | `172.16.201.0/24` |
| VLAN 202 | DATA | `172.16.202.0/24` |

#### Branche 2 (CE22)

| VLAN | Nom | Réseau |
|------|-----|--------|
| VLAN 203 | Management | `172.16.203.0/24` |
| VLAN 204 | DATA | `172.16.204.0/24` |

### Redondance HSRP

```
VLAN 10 / 15 / 20 :
  ├── IP Virtuelle (Gateway)   →  1ère adresse du sous-réseau
  ├── Fédérateur1 (Actif)      →  2ème adresse
  └── Fédérateur2 (Passif)     →  3ème adresse
```

### Services configurés

- **EtherChannel** (mode TRUNK) : agrégation de liens entre Fédérateurs
- **DHCP** : sur Fédérateur1 & 2 pour VLAN 10/15 ; sur CE12/CE22 pour les branches
- **OSPF** :
  - `passive-interface default` sur tous les Fédérateurs
  - Exception : interface VLAN 300 (coût OSPF = 3000)
- **Liens Trunk** : VLAN natif 20, VLANs autorisés 10, 15, 20, 300

---

## 🔒 Chapitre 3 — Monitoring, AAA & Sécurité pfSense

### 3.1 — Supervision avec Zabbix

**Serveur Zabbix** : `172.16.220.250/24` (Ubuntu Server, réseau de Management VLAN 20)

#### Composants déployés

| Composant | Détail |
|-----------|--------|
| Base de données | MySQL |
| Serveur web | Apache + PHP |
| Agents | Déployés sur tous les équipements |
| Protocole | SNMPv3 (authentification MD5) |

#### Équipements supervisés

- Routeurs CE : CE11, CE12, CE21, CE22
- Commutateurs Layer 2 : SW1, SW2, SW3, SW4
- Commutateurs Layer 3 : Fédérateur1, Fédérateur2

#### Configuration SNMPv3

```cisco
snmp-server community SSIR RO
snmp-server group TEKUP v3 auth
snmp-server group MYGROUP v3 auth
snmp-server host 172.16.220.250 version 3 auth TEKUP
snmp-server user TEKUP MYGROUP v3 auth md5 SSIR
```

#### Services complémentaires

```cisco
! Syslog centralisé
logging on
logging 172.16.220.250
logging trap warning

! Synchronisation NTP
ntp server 172.16.220.250

! Sondes IP SLA (Branches → Siège)
ip sla 11
  udp-jitter 192.168.1.2 frequency 300
ip sla schedule 11 start-time now life forever
```

---

### 3.2 — Service AAA avec FreeRADIUS

**Serveur FreeRADIUS** : `172.16.220.200/24` (Ubuntu Server, réseau de Management VLAN 20)

#### Méthodes d'authentification

| Méthode | Description |
|---------|-------------|
| **Méthode 1** | Serveur RADIUS — `172.16.220.200` |
| **Méthode 2** | Local (fallback) |
| **Clé partagée** | `VPN_Cyber` |

#### Niveaux de privilèges

```
Privilège  1 → Accès lecture seule
Privilège 10 → Accès intermédiaire
Privilège 15 → Accès administrateur complet
```

#### Configuration sur les équipements

```cisco
aaa new-model
radius-server host 172.16.220.200 key VPN_Cyber
aaa authentication login default group radius local
aaa authorization exec default group radius local
```

---

### 3.3 — Sécurité pfSense (Pare-feu + DMZ)

**pfSense 2.7.0** déployé comme pare-feu périmétrique unique avec trois zones réseau.

#### Architecture de sécurité

```
                    ┌──────────────────────┐
                    │        pfSense        │
                    │   (pare-feu unique)   │
          ┌─────────┤  WAN │ LAN │ OPT2    ├──────────┐
          │         └──────┴─────┴─────────┘          │
          │                  │                         │
     CE11 G1/0          Fédérateur1              Zone DMZ
  192.168.1.2/30     172.16.220.0/24          172.16.255.0/25
  (Backbone MPLS)   (Réseau Management)
```

#### Interfaces pfSense

| Interface | Connexion | Adresse IP |
|-----------|-----------|------------|
| **WAN** (OPT1) | Routeur CE11 | `192.168.1.1/30` |
| **LAN** | Fédérateur1 | `172.16.220.1/24` |
| **OPT2 (DMZ)** | Zone DMZ | `172.16.255.1/25` |

#### Zone DMZ

| Serveur | Service | Réseau |
|---------|---------|--------|
| Serveur SMTP | Messagerie | `172.16.255.0/25` |
| Serveur DNS | Résolution de noms | `172.16.255.0/25` |
| Serveur WWW | Web | `172.16.255.0/25` |

> **Passerelle DMZ :** `172.16.255.1` (interface OPT2 de pfSense)

#### Politique de sécurité appliquée

```
✅ AUTORISÉ  :  Siège (VLAN10/15) + Branches (VLAN202/204) → DMZ (SMTP/DNS/WWW)
✅ AUTORISÉ  :  Admin1 (172.16.220.100) + Admin2 (172.16.220.150) → DMZ (HTTP/HTTPS/SSH/Telnet/ICMP)
❌ INTERDIT  :  LAN Siège ↔ LAN Branches (communications directes)
```

#### OSPF sur pfSense

```
Area 11 configurée sur les trois interfaces :
  ├── LAN   → 172.16.220.0/24  (vers Fédérateur1)
  ├── OPT1  → 192.168.1.0/30   (vers CE11)
  └── OPT2  → 172.16.255.0/25  (vers Zone DMZ)
```

---

## 📊 Plan d'adressage complet

### Backbone IP/MPLS

| Lien | Réseau | Adresse A | Adresse B |
|------|--------|-----------|-----------|
| PE1 – P1 | `10.1.1.0/30` | PE1 G1/0 = `.1` | P1 G2/0 = `.2` |
| PE1 – P2 | `10.1.1.4/30` | PE1 G2/0 = `.5` | P2 G3/0 = `.6` |
| PE2 – P2 | `10.1.1.8/30` | PE2 G1/0 = `.9` | P2 G2/0 = `.10` |
| PE2 – P1 | `10.1.1.12/30` | PE2 G2/0 = `.13` | P1 G3/0 = `.14` |
| P1 – P2 | `10.1.1.20/30` | P1 G1/0 = `.21` | P2 G1/0 = `.22` |

### Loopbacks

| Routeur | Loopback0 |
|---------|-----------|
| PE1 | `1.1.1.1/32` |
| PE2 | `2.2.2.2/32` |
| P1 | `3.3.3.3/32` |
| P2 | `4.4.4.4/32` |
| CE11 | `172.16.11.11/32` |
| CE12 | `172.16.12.12/32` |
| CE21 | `172.16.21.21/32` |
| CE22 | `172.16.22.22/32` |

### Liens WAN CE ↔ PE (VRF CYBER)

| Lien | Réseau | CE (G1/0) | PE (G3/0 ou G4/0) |
|------|--------|-----------|-------------------|
| CE11 – PE1 | `192.168.1.0/30` | `.2` | PE1 G3/0 = `.1` |
| CE21 – PE1 | `192.168.1.4/30` | `.6` | PE1 G4/0 = `.5` |
| CE12 – PE2 | `192.168.1.8/30` | `.10` | PE2 G3/0 = `.9` |
| CE22 – PE2 | `192.168.1.12/30` | `.14` | PE2 G4/0 = `.13` |

> ⚠️ **Important :** La plage `192.168.1.0/24` est **entièrement réservée** aux liens WAN CE-PE du Backbone MPLS.

### Équipements de management (VLAN 20)

| Équipement | Adresse IP |
|-----------|-----------|
| Fédérateur1 | `172.16.220.2/24` |
| Fédérateur2 | `172.16.220.3/24` |
| SW1 | `172.16.220.101/24` |
| SW2 | `172.16.220.102/24` |
| Admin1 | `172.16.220.100/24` |
| Admin2 | `172.16.220.150/24` |
| Serveur AAA (FreeRADIUS) | `172.16.220.200/24` |
| Serveur Monitoring (Zabbix) | `172.16.220.250/24` |
| pfSense (interface LAN) | `172.16.220.1/24` |

### Zone DMZ

| Équipement | Réseau | Passerelle |
|-----------|--------|-----------|
| Serveurs SMTP / DNS / WWW | `172.16.255.0/25` | `172.16.255.1` (pfSense OPT2) |

---

## 🛠️ Technologies utilisées

### Simulation & Virtualisation

| Outil | Usage |
|-------|-------|
| **GNS3** | Émulation des routeurs et commutateurs Cisco |
| **GNS3 VM** | Optimisation des performances de simulation |
| **VMware** | Hébergement des serveurs (Zabbix, FreeRADIUS, pfSense) |

### Équipements réseau

| Type | Modèle | Usage |
|------|--------|-------|
| Routeurs | Cisco C700 | CE11, CE12, CE21, CE22, PE1, PE2, P1, P2 |
| Switch Layer 3 | Cisco 3650-24PS | Fédérateur1, Fédérateur2 |
| Switch Layer 2 | Cisco 2960-24TT | SW1, SW2, SW3, SW4 |

### Logiciels & Services

| Logiciel | Version | Rôle |
|---------|---------|------|
| **pfSense** | 2.7.0 | Pare-feu périmétrique (WAN/LAN/DMZ) |
| **Zabbix** | LTS | Supervision réseau centralisée (SNMPv3) |
| **FreeRADIUS** | 3.0 | Serveur AAA (authentification/autorisation) |
| **MySQL** | — | Base de données Zabbix |
| **Ubuntu Server** | — | OS des serveurs (Zabbix, FreeRADIUS, DMZ) |
| **Apache + PHP** | — | Interface web Zabbix |

### Protocoles implémentés

| Protocole | Couche | Usage |
|-----------|--------|-------|
| **OSPF** | Réseau | Routage dynamique — Backbone (Area 0) + Sites |
| **MP-BGP** | Application | Échange de routes VPN entre PE |
| **LDP** | Réseau | Distribution des labels MPLS |
| **VRF** | Réseau | Isolation du trafic client (VRF CYBER) |
| **HSRP** | Réseau | Redondance de passerelle sur les Fédérateurs |
| **EtherChannel** | Liaison | Agrégation de liens (Trunk) entre Fédérateurs |
| **SNMPv3** | Application | Supervision réseau avec authentification |
| **RADIUS** | Application | Authentification centralisée AAA |
| **SSH v2** | Application | Accès sécurisé aux équipements |
| **NTP** | Application | Synchronisation temporelle centralisée |
| **Syslog** | Application | Journalisation centralisée vers Zabbix |
| **IP SLA** | Application | Mesure de latence/jitter entre sites |

---

## 📁 Structure du dépôt

```
📦 secure-ip-mpls-backbone/
│
├── 📄 README.md                     ← Ce fichier
├── 📄 rapport_corrige.tex           ← Code source LaTeX du rapport complet
│
└── 📂 figures/                      ← Captures d'écran des configurations
    ├── fig01-architecture-vpn-mpls.*
    ├── fig03-topologie-entreprise.*
    ├── fig04-interfaces-PE1.*
    │   ...
    ├── fig65-interface-admin-zabbix.*
    ├── fig66-verification-snmpv3-SW4.*
    ├── fig70-ospf-networks-pfsense.*
    └── fig71-ospf-interfaces-pfsense.*
```

---

## 📝 Comment compiler le rapport

### Prérequis

- **LaTeX** : TeX Live 2022+ ou MiKTeX
- **Packages requis** : `babel` (french), `graphicx`, `longtable`, `mdframed`, `listings`, `fancyhdr`, `hyperref`

### Compilation

```bash
# Cloner le dépôt
git clone https://github.com/<votre-username>/secure-ip-mpls-backbone.git
cd secure-ip-mpls-backbone

# Compiler le rapport (2 passes pour la table des matières)
pdflatex rapport_corrige.tex
pdflatex rapport_corrige.tex

# Ou avec latexmk (recommandé)
latexmk -pdf rapport_corrige.tex
```

### Structure du rapport LaTeX

| Fichier image attendu | Figure | Description |
|-----------------------|--------|-------------|
| `fig01-architecture-vpn-mpls.*` | Fig. 1.1 | Architecture VPN-MPLS |
| `fig03-topologie-entreprise.*` | Fig. 1.3 | Topologie réseau complète |
| `fig65-interface-admin-zabbix.*` | Fig. 3.25 | Dashboard Zabbix |
| `fig70-ospf-networks-pfsense.*` | Fig. 3.30 | OSPF sur pfSense (Area 11) |
| `fig71-ospf-interfaces-pfsense.*` | Fig. 3.31 | Interfaces OSPF pfSense |

> Placez toutes vos captures d'écran dans le même répertoire que le fichier `.tex`, avec les noms exacts indiqués ci-dessus.

---

## 📌 Points clés du projet

- ✅ **Backbone IP/MPLS** complet avec redondance double sur chaque PE
- ✅ **VRF CYBER** déployée sur PE1 et PE2 pour l'isolation du trafic
- ✅ **LAN étendu** avec HSRP, EtherChannel et DHCP sur 4 sites
- ✅ **Supervision centralisée** Zabbix avec SNMPv3, Syslog et NTP
- ✅ **AAA** FreeRADIUS avec 3 niveaux de privilèges et fallback local
- ✅ **pfSense** séparant 3 zones : WAN (CE11), LAN (Fédérateur1), DMZ (SMTP/DNS/WWW)
- ✅ **OSPF Area 11** commun à pfSense, CE11 et Fédérateurs pour la continuité de routage
- ✅ **Politique de sécurité** : accès DMZ autorisé depuis Siège + Branches, communication inter-LAN bloquée

---

## 📚 Références

- RFC 3031 — MPLS Architecture
- RFC 4364 — BGP/MPLS IP VPNs
- RFC 2328 — OSPF Version 2
- RFC 2865 — Remote Authentication Dial In User Service (RADIUS)
- [Documentation pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Documentation Zabbix](https://www.zabbix.com/documentation/current/)
- [Documentation FreeRADIUS](https://freeradius.org/documentation/)

---

<div align="center">

**TEK-UP University — SSIR-4-A — 2025/2026**  
*Projet encadré par Mr. Tarek HDIJI*

![GitHub](https://img.shields.io/badge/GitHub-Rapport_LaTeX-181717?style=flat-square&logo=github)

</div>
