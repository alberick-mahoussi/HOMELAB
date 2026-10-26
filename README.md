# 🛡️ SOC Analyst — Advanced Home Lab

> **Learn • Detect • Investigate • Respond**
> Un environnement complet d'entraînement SOC (Security Operations Center) construit sous **Oracle VirtualBox**, intégrant SIEM, détection réseau, Active Directory, machines vulnérables et outils d'attaque.

![Platform](https://img.shields.io/badge/Platform-VirtualBox-blue)
![SIEM](https://img.shields.io/badge/SIEM-Splunk%20%7C%20Wazuh-green)
![NSM](https://img.shields.io/badge/NSM-Security%20Onion-purple)
![Status](https://img.shields.io/badge/Status-Lab-orange)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Table des matières

1. [Objectifs du lab](#-objectifs-du-lab)
2. [Architecture](#-architecture)
3. [Prérequis matériel & logiciel](#-prérequis-matériel--logiciel)
4. [Plan d'adressage réseau](#-plan-dadressage-réseau)
5. [Configuration réseau VirtualBox](#-configuration-réseau-virtualbox)
6. [Installation des machines virtuelles](#-installation-des-machines-virtuelles)
    - [pfSense (Firewall / Routeur)](#1-pfsense--firewall--routeur)
    - [Windows Server (Active Directory)](#2-windows-server--active-directory)
    - [Windows 11 Workstation](#3-windows-11-workstation)
    - [Ubuntu Server](#4-ubuntu-server)
    - [Kali Linux](#5-kali-linux-attaquant)
    - [Metasploitable 2](#6-metasploitable-2-cible-vulnérable)
    - [Wazuh](#7-wazuh--siem--xdr--hids)
    - [Splunk Enterprise](#8-splunk-enterprise--siem)
    - [Security Onion](#9-security-onion--nsm)
7. [Ports & protocoles](#-ports--protocoles)
8. [Scénarios d'entraînement](#-scénarios-dentraînement-use-cases)
9. [Checklist de validation](#-checklist-de-validation)
10. [Dépannage](#-dépannage-troubleshooting)
11. [Ressources](#-ressources)

---

## Objectifs du lab

Ce home lab reproduit un environnement d'entreprise réaliste pour s'entraîner en tant qu'analyste SOC niveau 1 :

- **Learn** — Comprendre l'architecture réseau, les logs et la télémétrie de sécurité.
- **Detect** — Configurer des SIEM (Splunk, Wazuh) et de la détection réseau (Security Onion / Suricata).
- **Investigate** — Analyser des logs, du PCAP, corréler des alertes.
- **Respond** — Pratiquer l'investigation (DFIR), le threat hunting et la réponse à incident.

---

## Architecture

![Architecture globale du lab](docs/screenshots/00-architecture.png)
>  *Remplace cette image par le schéma de ton architecture.*

| Machine | Rôle | OS | RAM conseillée |
|---|---|---|---|
| **pfSense** | Firewall / NAT / Routing / VPN | FreeBSD | 1 Go |
| **Windows Server** | Active Directory / DNS / DHCP | Windows Server 2022 | 4 Go |
| **Windows 11** | Poste de travail (cible + télémétrie) | Windows 11 | 4 Go |
| **Ubuntu Server** | Serveur Linux (logs, services) | Ubuntu 22.04 | 2 Go |
| **Kali Linux** | Machine attaquante | Kali Rolling | 2–4 Go |
| **Metasploitable 2** | Cible volontairement vulnérable | Debian (old) | 512 Mo |
| **Wazuh** | SIEM / XDR / HIDS | Ubuntu 22.04 | 4 Go |
| **Splunk Enterprise** | SIEM (ingestion, corrélation, dashboards) | Ubuntu 22.04 | 4–8 Go |
| **Security Onion** | NSM (PCAP, IDS/IPS, visibilité réseau) | Security Onion 2.4 | 8–12 Go |

---

## Prérequis matériel & logiciel

### Matériel (hôte)
- **CPU** : 8 cœurs avec virtualisation activée (VT-x / AMD-V dans le BIOS).
- **RAM** : **32 Go minimum** recommandé (16 Go possible en démarrant les VM par phases).
- **Disque** : **500 Go SSD** libres minimum.
- Virtualisation imbriquée activée pour certaines VM si besoin.


### Logiciel
- [Oracle VirtualBox](https://www.virtualbox.org/) (dernière version)
- [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads)
- ISOs / appliances des différents OS (liens en [Ressources](#-ressources))

---

## Plan d'adressage réseau

Réseau du lab : **`10.0.0.0/24`** (interne, isolé derrière pfSense).

| Machine | IP fixe | Interface |
|---|---|---|
| pfSense (LAN) | `10.0.0.1` | Interne `LAB-NET` |
| pfSense (WAN) | DHCP (NAT) | NAT VirtualBox |
| Windows Server (AD/DNS/DHCP) | `10.0.0.10` | `LAB-NET` |
| Windows 11 Workstation | `10.0.0.20` | `LAB-NET` |
| Ubuntu Server | `10.0.0.30` | `LAB-NET` |
| Kali Linux | `10.0.0.40` | `LAB-NET` |
| Metasploitable 2 | `10.0.0.50` | `LAB-NET` |
| Wazuh | `10.0.0.60` | `LAB-NET` |
| Splunk Enterprise | `10.0.0.70` | `LAB-NET` |
| Security Onion (mgmt) | `10.0.0.80` | `LAB-NET` |
| Security Onion (sniff) | — | Interface en mode promiscuous |

>  **Important** : la plage `10.0.0.0/24` doit rester **isolée**. Seul pfSense a accès à Internet via le NAT. Metasploitable 2 ne doit **jamais** être exposé directement au WAN.

---

##  Configuration réseau VirtualBox

Le lab utilise une combinaison de modes réseau :

| Usage | Mode VirtualBox | Détail |
|---|---|---|
| Accès Internet (pfSense WAN) | **NAT** | Fournit l'accès sortant au lab |
| Réseau interne du lab | **Internal Network** nommé `LAB-NET` | Isolation totale |
| Mirroring Security Onion | **Promiscuous : Allow All** | Capture du trafic (SPAN/TAP) |

### Créer le réseau interne

1. Dans chaque VM → **Settings → Network**.
2. Adapter 1 : `Internal Network` → nom : **`LAB-NET`**.
3. Pour pfSense uniquement : Adapter 1 = **NAT** (WAN), Adapter 2 = **Internal Network `LAB-NET`** (LAN).
4. Pour Security Onion : activer **Promiscuous Mode → Allow All** sur l'interface de capture.

![Configuration réseau VirtualBox](docs/screenshots/01-vbox-network.png)
> 📸 *Screenshot : onglet Network d'une VM.*

---

##  Installation des machines virtuelles

> 💡 **Convention screenshots** : place tes captures dans `docs/screenshots/` et garde la numérotation proposée pour une doc propre.

---

### 1. pfSense — Firewall / Routeur

**Rôle** : passerelle du lab, NAT, règles firewall, DHCP optionnel, VPN.

**Étapes :**
1. Crée une VM : Type **BSD**, Version **FreeBSD (64-bit)**, 1 Go RAM, 16 Go disque.
2. Network : Adapter 1 = **NAT**, Adapter 2 = **Internal `LAB-NET`**.
3. Boote sur l'ISO pfSense et installe (assignation : WAN = `em0`, LAN = `em1`).
4. Configure l'IP LAN sur **`10.0.0.1/24`**.
5. Accède à l'interface web `https://10.0.0.1` depuis une VM du LAN.
6. Crée les règles firewall (autorise le LAN, logge les refus).

![pfSense - assignation des interfaces](docs/screenshots/02-pfsense-interfaces.png)
![pfSense - interface web](docs/screenshots/03-pfsense-webgui.png)

**Règles firewall recommandées :**
- Autoriser `LAB-NET → WAN` (navigation, mises à jour).
- Activer le **logging** sur les règles de blocage (utile pour corréler dans le SIEM).

---

### 2. Windows Server — Active Directory

**Rôle** : contrôleur de domaine, DNS, DHCP, source de logs AD.

**Étapes :**
1. VM : Windows Server 2022, 4 Go RAM, 60 Go disque, réseau `LAB-NET`.
2. IP fixe : `10.0.0.10`, passerelle `10.0.0.1`, DNS `127.0.0.1`.
3. Installe le rôle **AD DS** → promeus en contrôleur de domaine.
   - Nom de domaine : ex. `soclab.local`
4. Installe les rôles **DNS** et **DHCP** (ou laisse DHCP à pfSense).
5. Crée des **OU**, **utilisateurs** et **GPO** de test.
6. Installe **Sysmon** (config [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config)) pour une télémétrie riche.

![Promotion contrôleur de domaine](docs/screenshots/04-ad-promotion.png)
![Utilisateurs et OU AD](docs/screenshots/05-ad-users.png)

**Logs à collecter :** `Security`, `System`, `Sysmon/Operational`, logs AD.

---

### 3. Windows 11 Workstation

**Rôle** : poste utilisateur réaliste, cible d'attaques, source de logs endpoint.

**Étapes :**
1. VM : Windows 11, 4 Go RAM, 60 Go disque, `LAB-NET`.
2. IP fixe `10.0.0.20`, DNS = `10.0.0.10` (le DC).
3. **Rejoins le domaine** `soclab.local`.
4. Installe **Sysmon** + l'**agent Wazuh** + le **Universal Forwarder Splunk**.
5. Active l'audit avancé (PowerShell logging, process creation, etc.).

![Jonction au domaine](docs/screenshots/06-win11-domain-join.png)
![Sysmon installé](docs/screenshots/07-win11-sysmon.png)

---

### 4. Ubuntu Server

**Rôle** : serveur Linux générique (web, SSH), source de logs système.

**Étapes :**
1. VM : Ubuntu Server 22.04, 2 Go RAM, 25 Go disque, `LAB-NET`, IP `10.0.0.30`.
2. Installe et configure **auditd** pour l'audit système.
3. Installe l'**agent Wazuh** et/ou configure le **forwarding syslog**.
4. (Optionnel) héberge un service web vulnérable pour l'entraînement.

```bash
# Exemple : installation auditd
sudo apt update && sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

![Ubuntu auditd](docs/screenshots/08-ubuntu-auditd.png)

---

### 5. Kali Linux (Attaquant)

**Rôle** : machine offensive pour générer du trafic d'attaque (scans, exploits, brute force, attaques web).

**Étapes :**
1. VM : Kali Linux, 2–4 Go RAM, `LAB-NET`, IP `10.0.0.40`.
2. Mets à jour : `sudo apt update && sudo apt full-upgrade -y`.
3. Outils déjà présents : `nmap`, `metasploit`, `hydra`, `nikto`, `burpsuite`, etc.

> ⚠️ **Usage strictement limité au lab isolé.** N'utilise ces outils que contre tes propres machines (`10.0.0.0/24`).

![Kali - nmap scan](docs/screenshots/09-kali-nmap.png)

---

### 6. Metasploitable 2 (Cible vulnérable)

**Rôle** : cible d'entraînement contenant de nombreuses vulnérabilités connues.

**Étapes :**
1. Importe l'appliance **Metasploitable 2**.
2. Réseau : **`LAB-NET` uniquement**, IP `10.0.0.50`.
3. Identifiants par défaut : `msfadmin / msfadmin`.

>  **Jamais exposer au WAN.** Cette machine est volontairement non sécurisée.

![Metasploitable 2 login](docs/screenshots/10-metasploitable.png)

---

### 7. Wazuh — SIEM / XDR / HIDS

**Rôle** : collecte de logs, détection de menaces, FIM (file integrity monitoring), réponse à incident.

**Étapes :**
1. VM : Ubuntu 22.04, 4 Go RAM, 50 Go disque, `LAB-NET`, IP `10.0.0.60`.
2. Installation tout-en-un :

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

3. Récupère le mot de passe `admin` affiché en fin d'installation.
4. Accède au dashboard : `https://10.0.0.60`.
5. Déploie les **agents Wazuh** sur Windows 11, Windows Server et Ubuntu (port **1514/UDP**).

![Wazuh dashboard](docs/screenshots/11-wazuh-dashboard.png)
![Wazuh agents connectés](docs/screenshots/12-wazuh-agents.png)

---

### 8. Splunk Enterprise — SIEM

**Rôle** : ingestion, recherche, corrélation, dashboards, alertes.

**Étapes :**
1. VM : Ubuntu 22.04, 4–8 Go RAM, 80 Go disque, `LAB-NET`, IP `10.0.0.70`.
2. Installe Splunk Enterprise (version gratuite jusqu'à 500 Mo/jour) :

```bash
# Après téléchargement du .deb depuis splunk.com
sudo dpkg -i splunk-*.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

3. Interface web : `http://10.0.0.70:8000`.
4. Configure un **receiver** sur le port **9997/TCP** pour les forwarders.
5. Installe le **Universal Forwarder** sur les endpoints, ou configure le **syslog** (514/UDP).
6. Ajoute les apps : *Splunk Add-on for Microsoft Windows*, *Sysmon*, etc.

![Splunk - data inputs](docs/screenshots/13-splunk-inputs.png)
![Splunk - dashboard de recherche](docs/screenshots/14-splunk-search.png)

---

### 9. Security Onion — NSM

**Rôle** : détection réseau, capture PCAP, IDS/IPS (Suricata), visibilité réseau.

**Étapes :**
1. VM : 8–12 Go RAM, 200 Go disque, 2 interfaces :
   - Management : `LAB-NET`, IP `10.0.0.80`.
   - Sniffing : interface en **mode promiscuous (Allow All)**.
2. Installe depuis l'ISO Security Onion 2.4 (choisir le mode **Standalone**).
3. Configure l'interface de monitoring pour recevoir le **trafic miroir (SPAN/TAP)**.
4. Accède à la console web et explore **Alerts**, **Hunt**, **PCAP**.


![Security Onion - alerts](docs/screenshots/15-securityonion-alerts.png)
![Security Onion - PCAP](docs/screenshots/16-securityonion-pcap.png)

---

##  Ports & protocoles

| Service | Port | Protocole | Usage dans le lab |
|---|---|---|---|
| HTTP | 80 | TCP | Trafic web / attaques web |
| HTTPS | 443 | TCP | Interfaces d'administration |
| SSH | 22 | TCP | Accès Linux |
| RDP | 3389 | TCP | Accès Windows |
| SMB | 445 | TCP | Partages Windows / latéralisation |
| DNS | 53 | TCP/UDP | Résolution de noms |
| DHCP | 67/68 | UDP | Attribution d'adresses |
| Syslog | 514 | UDP | Forwarding de logs |
| Wazuh agent | 1514 | UDP | Agents → Wazuh |
| Splunk forwarder | 9997 | TCP | Forwarders → Splunk |

---

##  Dépannage (Troubleshooting)

| Problème | Cause probable | Solution |
|---|---|---|
| Pas d'Internet dans le LAB-NET | Règle pfSense ou NAT mal configuré | Vérifier l'adapter NAT (WAN) de pfSense |
| Les VM ne se voient pas | Mauvais nom de réseau interne | Vérifier que toutes sont sur `LAB-NET` |
| Agent Wazuh déconnecté | Port 1514/UDP bloqué | Ouvrir le port côté firewall/host |
| Splunk ne reçoit rien | Receiver 9997 non activé | `splunk enable listen 9997` |
| Security Onion ne capture pas | Promiscuous mode désactivé | Mettre l'interface en `Allow All` |
| VM lentes | RAM insuffisante | Démarrer les VM par phases |

---

##  Ressources

- [pfSense](https://www.pfsense.org/)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Splunk Free](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- [Security Onion](https://securityonionsolutions.com/)
- [Sysmon config (SwiftOnSecurity)](https://github.com/SwiftOnSecurity/sysmon-config)
- [Metasploitable 2](https://sourceforge.net/projects/metasploitable/)
- [MITRE ATT&CK](https://attack.mitre.org/)

---

##  Organisation des screenshots

```
docs/
└── screenshots/
    ├── 00-architecture.png
    ├── 01-vbox-network.png
    ├── 02-pfsense-interfaces.png
    ├── 03-pfsense-webgui.png
    ├── 04-ad-promotion.png
    ├── 05-ad-users.png
    ├── 06-win11-domain-join.png
    ├── 07-win11-sysmon.png
    ├── 08-ubuntu-auditd.png
    ├── 09-kali-nmap.png
    ├── 10-metasploitable.png
    ├── 11-wazuh-dashboard.png
    ├── 12-wazuh-agents.png
    ├── 13-splunk-inputs.png
    ├── 14-splunk-search.png
    ├── 15-securityonion-alerts.png
    ├── 16-securityonion-pcap.png
    └── 17-usecase-bruteforce.png
```

---

## ⚖️ Avertissement

Ce lab est destiné **exclusivement** à l'apprentissage dans un **environnement isolé**. Les outils offensifs (Kali) et les machines vulnérables (Metasploitable) ne doivent **jamais** être utilisés contre des systèmes que tu ne possèdes pas ou sans autorisation explicite.

---

## 📄 Licence

Distribué sous licence MIT. Voir `LICENSE` pour plus d'informations.

---

> ⭐ Si ce lab t'aide, n'hésite pas à star le repo et à partager tes propres scénarios !
