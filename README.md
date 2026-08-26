# 🛡️ Architecture Réseau Haute Disponibilité & Sécurité — FortiGate HA under EVE-NG

> **Lab de cybersécurité réseau** — Infrastructure d'entreprise complète avec redondance multi-niveaux sur EVE-NG.  
> Cluster FortiGate Actif/Passif · HSRP · VLANs · DMZ · Failover testé et validé ✅

![EVE-NG](https://img.shields.io/badge/Platform-EVE--NG-blue?style=for-the-badge&logo=linux)
![FortiGate](https://img.shields.io/badge/Firewall-FortiGate_VM64-red?style=for-the-badge)
![Cisco](https://img.shields.io/badge/Routing-Cisco_IOS-1BA0D7?style=for-the-badge&logo=cisco)
![HA](https://img.shields.io/badge/Mode-HA_Actif%2FPassif-brightgreen?style=for-the-badge)
![Status](https://img.shields.io/badge/Lab-Validé_✅-success?style=for-the-badge)

---

## 📌 Présentation du Projet

Ce projet documente la conception, le déploiement et la validation d'une **infrastructure réseau d'entreprise complète** émulée sous EVE-NG. L'architecture implémente la redondance et la haute disponibilité à **trois niveaux distincts** :

| Niveau | Technologie | Équipements | Rôle |
|:---|:---|:---|:---|
| **Périmètre de sécurité** | FortiGate HA A/P | FGT-1 / FGT-2 | Firewall redondant |
| **Cœur de réseau L3** | HSRP | R1 (Master) / R2 (Backup) | Passerelle redondante |
| **Accès & segmentation L2** | VLANs 802.1Q | SW1 / SW2 | Isolation des segments |

### 🎯 Objectifs du Lab

- ✅ Déployer un cluster FortiGate en mode **Actif/Passif** avec synchronisation de configuration
- ✅ Valider le **failover automatique** sans interruption de service
- ✅ Configurer la **redondance HSRP** entre deux routeurs Cisco
- ✅ Segmenter le réseau en **5 VLANs** (Utilisateurs, Serveurs, DMZ, VoIP, Management)
- ✅ Provisionner un **serveur DHCP DMZ** sur le cluster FortiGate

---

## 📐 Topologie Réseau & Architecture

![Topologie EVE-NG](img/topo.png)

### Vue d'ensemble de l'architecture

```
                    ┌─────────────────────────────────────────────┐
         ISP1 ──────┤  FGT-1 (Primary)   HA A/P   FGT-2 (Backup) ├────── ISP2
                    │  port1↔port1 Heartbeat 169.254.0.0/24       │
                    │  port3 ──────── DMZ (VLAN30) ───────────────│
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │  R1 (HSRP Master)   R2 (HSRP Backup)        │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │     SW1              SW2     │
                    └──┬──────┬──────┬──────┬─────┘
                       │      │      │      │
                    VLAN10  VLAN20  VLAN40  VLAN50
                 Utilisat. Serveurs IP Phone Mgmt
```

---

## 🗺️ Plan d'Adressage IP & Segmentation VLAN

| Zone / VLAN | Nom | Réseau | Passerelle | Interface FW | Équipements |
|:---|:---|:---|:---|:---|:---|
| **HA Heartbeat** | Sync | `169.254.0.0/24` | — | `port1` (FGT-1↔FGT-2) | Lien dédié HA |
| **WAN/Mgmt FW** | WAN | DHCP / ISP | — | `port2` | HTTP · HTTPS · SSH |
| **VLAN 30 — DMZ** | DMZ | `192.168.30.0/24` | `192.168.30.254` | `port3` | Serveur Linux |
| **VLAN 10 — Users** | Utilisateurs | `192.168.10.0/24` | VIP HSRP | SW1 `e1/0-1` | VPC Windows · macOS · Linux |
| **VLAN 20 — Servers** | Serveurs | `192.168.20.0/24` | VIP HSRP | SW1 `e1/2-3` | Windows Server 2019 |
| **VLAN 40 — VoIP** | IP Phone | `192.168.40.0/24` | VIP HSRP | SW1/SW2 `e1/0` | Téléphones IP |
| **VLAN 50 — Mgmt** | Management | `192.168.50.0/24` | VIP HSRP | SW2 `e2/0` | Poste Admin |

### 📍 Interfaces FortiGate

| Interface | FGT-1 | FGT-2 | Rôle |
|:---|:---|:---|:---|
| `port1` | 169.254.0.1 | 169.254.0.2 | Heartbeat HA & Sync |
| `port2` | DHCP (ISP1) | DHCP (ISP2) | WAN · Management |
| `port3` | `192.168.30.254/24` | (sync) | DMZ Gateway + DHCP |
| `port4` | — | — | Réservé |

---

## ⚙️ Guide de Configuration

### 1️⃣ Configuration des Interfaces FortiGate (CLI FortiOS)

```bash
# === INTERFACE port3 — DMZ ===
config system interface
    edit "port3"
        set alias "DMZ-VLAN30"
        set ip 192.168.30.254 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
end
```

### 2️⃣ Serveur DHCP sur la zone DMZ

```bash
config system dhcp server
    edit 1
        set dns-service default
        set interface "port3"
        set default-gateway 192.168.30.254
        set netmask 255.255.255.0
        config ip-range
            edit 1
                set start-ip 192.168.30.10
                set end-ip   192.168.30.50
            next
        end
    next
end
```

### 3️⃣ Configuration HA — FGT-1 (Master / Priority 200)

```bash
config system ha
    set group-id    10
    set group-name  "LAB-HA-CLUSTER"
    set mode        a-p
    set hbdev       "port1" 50
    set override    disable
    set priority    200
    set session-pickup enable
    set session-pickup-connectionless enable
end
```

### 4️⃣ Configuration HA — FGT-2 (Slave / Priority 100)

```bash
config system ha
    set group-id    10
    set group-name  "LAB-HA-CLUSTER"
    set mode        a-p
    set hbdev       "port1" 50
    set override    disable
    set priority    100
    set session-pickup enable
    set session-pickup-connectionless enable
end
```

> ⚠️ **Important** : Le `group-id` et le `group-name` doivent être **strictement identiques** sur les deux nœuds. Seule la `priority` diffère.

### 5️⃣ Configuration HSRP — R1 (Master)

```bash
interface GigabitEthernet0/1
 description "Lien vers SW1/SW2 - VLAN Trunk"
 ip address 192.168.1.1 255.255.255.0
 standby 1 ip 192.168.1.254        ! VIP partagée
 standby 1 priority 110            ! > 100 = Master
 standby 1 preempt
 standby 1 track GigabitEthernet0/0 20
 no shutdown
```

### 6️⃣ Configuration HSRP — R2 (Backup)

```bash
interface GigabitEthernet0/1
 ip address 192.168.1.2 255.255.255.0
 standby 1 ip 192.168.1.254        ! Même VIP
 standby 1 priority 100            ! < 110 = Backup
 standby 1 preempt
 no shutdown
```

---

## 🔬 Validation & Tests de Haute Disponibilité

### Vérification de l'état du cluster HA

```bash
FGT-1 # get system ha status
```

![Résultat HA Status](img/resultaFailhover.png)

**Points de validation clés :**

| Paramètre | Valeur observée | Signification |
|:---|:---|:---|
| `HA Health Status` | `OK` | Cluster en bonne santé |
| `Mode` | `HA A-P` | Actif/Passif confirmé |
| `Group` | `10` | Group-ID correct |
| `Configuration Status` | `in-sync` (×2) | Synchro bidirectionnelle OK ✅ |
| `Primary` | `FGT-2` (post-failover) | Bascule effectuée avec succès |
| `Secondary` | `FGT-1` | Ancien master rétrogradé |

### Vérification de la synchronisation des checksums

```bash
FGT-1 # diagnose sys ha checksum cluster
```

> Les checksums doivent être **identiques** sur les deux membres. Toute différence indique un problème de synchronisation.

---

## 🔀 Test de Failover — Bascule Forcée

![Exécution du Failover](img/executionfailhover.png)

### Procédure de test

```bash
# Sur FGT-1 (actuellement Primary) — forcer la bascule
FGT-1 # execute ha failover set 1

Caution: This command will trigger an HA failover.
It is intended for testing purposes.
Do you want to continue? (y/n) y
```

### Chronologie de la bascule (extrait des logs)

```
[13:39:36] FGVMEVVAWYBEQSD8 selected as primary — only member in cluster
[13:41:52] FGVMEV2Q_WM81RB3 selected as primary — uptime larger than peer
[13:51:45] FGVMEVVAWYBEQSD8 selected as primary — EXE_FAIL_OVER flag set on peer
```

### Analyse de la séquence d'élection

```
T+0s  ──▶ FGT-1 exécute "execute ha failover set 1"
           Le flag EXE_FAIL_OVER est positionné sur FGT-1

T+~5s ──▶ FGT-2 détecte le flag via le lien Heartbeat (port1)
           Algorithme d'élection HA déclenché

T+~8s ──▶ FGT-2 prend le rôle Primary (FGVMEVVAWYBEQSD8)
           FGT-1 rétrogradé au rôle Secondary

T+~10s──▶ Synchronisation confirmée : "in-sync" sur les deux membres
           Reprise de service transparente ✅
```

### Résultats du test

| Critère de validation | Résultat |
|:---|:---|
| Bascule déclenchée sur commande | ✅ |
| FGT-2 élu Primary automatiquement | ✅ |
| Configuration synchronisée post-failover | ✅ `in-sync` |
| Sessions maintenues (session-pickup) | ✅ |
| Temps de bascule | **< 10 secondes** |
| Continuité de service DMZ | ✅ |

---

## 📁 Structure du Dépôt

```
📦 fortigate-ha-lab/
├── 📄 README.md                    ← Ce fichier
├── 📄 RAPPORT_TP.md                ← Rapport de TP complet
├── 📁 img/
│   ├── 🖼️  topo.png                ← Capture topologie EVE-NG
│   ├── 🖼️  resultaFailhover.png    ← get system ha status post-failover
│   └── 🖼️  executionfailhover.png  ← Exécution du test de bascule
├── 📁 configs/
│   ├── 📄 FGT-1-HA.conf            ← Config FortiGate Master
│   ├── 📄 FGT-2-HA.conf            ← Config FortiGate Slave
│   ├── 📄 R1-HSRP.txt              ← Config Routeur Master HSRP
│   └── 📄 R2-HSRP.txt              ← Config Routeur Backup HSRP
└── 📁 topologies/
    └── 📄 lab-fortigate-ha.unl     ← Topologie EVE-NG importable
```

---

## 🚀 Reproduire ce Lab

### Prérequis

| Composant | Version |
|:---|:---|
| EVE-NG Community | 5.0+ |
| FortiGate-VM64-KVM | 7.x |
| Cisco IOSv | 15.x |
| RAM hôte recommandée | 16 GB+ |

### Import rapide

```bash
# 1. Cloner le dépôt
git clone https://github.com/votre-username/fortigate-ha-lab.git
cd fortigate-ha-lab

# 2. Importer la topologie dans EVE-NG
scp topologies/lab-fortigate-ha.unl root@EVE_IP:/opt/unetlab/labs/
ssh root@EVE_IP "chown -R www-data:www-data /opt/unetlab/labs/"

# 3. Démarrer tous les nœuds depuis l'interface EVE-NG
# 4. Appliquer les configs depuis le dossier configs/
```

---

## 🧠 Concepts Clés

| Concept | Détail |
|:---|:---|
| **FortiGate HA A/P** | Un seul nœud traite le trafic. Le secondaire synchronise la config et prend le relais en cas de panne. |
| **Heartbeat HA** | Lien dédié (port1) sur `169.254.0.0/24` pour la détection de panne et la sync de session. |
| **HSRP** | Hot Standby Router Protocol — VIP partagée entre R1 et R2, bascule transparente pour les clients. |
| **Session Pickup** | Les sessions TCP actives sont maintenues lors du failover grâce à la synchronisation des tables de session. |
| **EXE_FAIL_OVER flag** | Flag positionné manuellement pour forcer une bascule en test, sans couper physiquement le nœud. |

---

## 📄 Licence & Auteur

> 🎓 Lab réalisé dans un cadre académique/professionnel — Infrastructure émulée sous EVE-NG.  
> ⚠️ À des fins éducatives uniquement.

![MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)
