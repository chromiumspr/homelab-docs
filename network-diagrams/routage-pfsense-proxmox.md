# Routage complet : Interfaces, routes et règles — v4.0

> **Mise à jour :** Architecture "Philosophie B" avec switch manageable Cisco SG350-28  
> **Remplace :** Version antérieure basée sur hub + `vmbr0` actif + LAN flat

---

## Inventaire des machines

| Machine | Rôle |
|---------|------|
| srv-proxmox-01 | Hyperviseur Proxmox, routeur NAT temporaire (vmbr1/vtnet0) |
| vm-pfsense-01 | Routeur/Firewall pfSense  |
| sw-backbone-01 | Switch  Cisco SG350-28 (`10.10.10.2/24`) |
| PC Admin | Poste d'administration (`10.10.10.7/24`) |
| Téléphone | Partage de connexion WiFi (internet tmp) |

---

## Interfaces réseau

### srv-proxmox-01

| Interface | Type | IP | Connecté à | Rôle |
|-----------|------|----|------------|------|
| nic0 | physique | — | sw-backbone-01 gi2 | Esclave de vmbr2 (trunk VLAN-aware) |
| wlp4s0 | WiFi | 172.20.10.10/28 (DHCP) | Téléphone | Internet temporaire |
| vmbr0 | bridge (`bridge-ports none`) | 192.168.10.10/24 | — | Inactif — just in case |
| vmbr1 | bridge virtuel point-à-point | 10.10.0.1/30 | vtnet0 pfSense | Lien WAN |
| vmbr2 | bridge virtuel VLAN-aware | `inet manual` (pas d'IP) | nic0 → switch gi2 | Trunk multi-VLAN |
| vmbr2.10 | sous-interface VLAN 10 | 10.10.10.10/24 | — | Accès management Proxmox |
| tap100i0 | tap VM | — | vmbr1 | Interface WAN VM pfSense |
| tap100i1 | tap VM | — | vmbr2 | Interface LAN/trunk VM pfSense |

> ⚠️ `vmbr2` doit rester `inet manual` (pas d'IP). L'accès management Proxmox passe **uniquement** par `vmbr2.10`.  
> ⚠️ `vmbr0` est conservé en structure mais inactif (`bridge-ports none`). 

### vm-pfsense-01

| Interface | Type | IP | Connecté à | Rôle |
|-----------|------|----|------------|------|
| vtnet0 | virtio | 10.10.0.2/30, gw 10.10.0.1 | vmbr1 Proxmox | WAN |
| vtnet1 | virtio trunk | pas d'IP directe | vmbr2 Proxmox | Trunk router-on-a-stick |
| vtnet1.10 | VLAN 10 | 10.10.10.1/24 | — | MGMT — gateway |
| vtnet1.20 | VLAN 20 | 10.10.20.1/24 | — | LAN — gateway |
| vtnet1.30 | VLAN 30 | 10.10.30.1/24 | — | SVC — gateway |
| vtnet1.40 | VLAN 40 | 10.10.40.1/24 | — | DMZ — gateway |
| vtnet1.50 | VLAN 50 | 10.10.50.1/24 | — | CYB — gateway |
| vtnet1.60 | VLAN 60 | 10.10.60.1/24 | — | FUNLAB — gateway |
| vtnet1.70 | VLAN 70 | 10.10.70.1/24 | — | BCKP — gateway |

### sw-backbone-01 (Cisco SG350-28)

| Port | Mode | VLAN(s) | Connecté à |
|------|------|---------|------------|
| gi2 | Trunk | VLANs 10–70 tagués, PVID natif VLAN 10 | Proxmox nic0 |
| gi7 | Access untagged | VLAN 10 | PC Admin |

IP management switch : `10.10.10.2/24`, gateway `10.10.10.1`

### PC Admin

| Interface | IP | Connecté à | Rôle |
|-----------|----|------------|------|
| Ethernet | 10.10.10.7/24 | sw-backbone-01 gi7 | Administration VLAN MGMT |

---

## Plan VLAN

| VLAN | Nom | Subnet | Gateway |
|------|-----|--------|---------|
| 10 | MGMT | 10.10.10.0/24 | 10.10.10.1 |
| 20 | LAN | 10.10.20.0/24 | 10.10.20.1 |
| 30 | SVC | 10.10.30.0/24 | 10.10.30.1 |
| 40 | DMZ | 10.10.40.0/24 | 10.10.40.1 |
| 50 | CYB | 10.10.50.0/24 | 10.10.50.1 |
| 60 | FUNLAB | 10.10.60.0/24 | 10.10.60.1 |
| 70 | BCKP | 10.10.70.0/24 | 10.10.70.1 |

DHCP activé par VLAN dans pfSense (plage `.100–.200`).

---

## Interconnexions

```
[Téléphone 172.20.10.1]
        │ WiFi 802.11 (temporaire)
        │
[wlp4s0 172.20.10.10] ← srv-proxmox-01
        │ NAT MASQUERADE
        │ ip rule table 100
        ▼
[vmbr1 10.10.0.1/30] ◄──────────────── [vtnet0 10.10.0.2/30]
                                                  │
                                           vm-pfsense-01
                                                  │
[vmbr2 inet manual] ◄──────────────── [vtnet1 trunk (no IP)]
        │                                vtnet1.10 … vtnet1.70
        │ (VLAN-aware bridge)
       nic0
        │ trunk 802.1Q (VLANs 10-70)
        ▼
[sw-backbone-01 gi2]
        │
  ┌─────┴──────┐
  gi7          …
  │
[PC Admin 10.10.10.7] (VLAN10 untagged)

[vmbr2.10 10.10.10.10/24] → accès Proxmox UI via VLAN MGMT
[vmbr0 192.168.10.10/24]  → INACTIF (bridge-ports none)
```

---

## Tables de routage

> **Lire :** "Tout ce qui vient de source vers destination sort par interface vers via"   

### Proxmox — table main

| Destination | Via | Interface | Source |
|-------------|-----|-----------|--------|
| default | 172.20.10.1 | wlp4s0 | — |
| 10.10.0.0/30 | — | vmbr1 | 10.10.0.1 |
| 10.10.10.0/24 | — | vmbr2.10 | 10.10.10.10 |
| 172.20.10.0/28 | — | wlp4s0 | 172.20.10.10 |
| 192.168.10.0/24 | — | vmbr0 | 192.168.10.10 |

> La default route pointe sur `wlp4s0` via un `post-up` dans `/etc/network/interfaces`. Elle ne doit **pas** être portée par `vmbr0` (interface linkdown → bug route disparue).

### Proxmox — table 100

| Destination | Via | Interface |
|-------------|-----|-----------|
| default | 172.20.10.1 | wlp4s0 |

Utilisée uniquement pour les paquets dont la source est `10.10.0.2` (pfSense WAN).

### Proxmox — ip rule

| Priorité | Règle | Table | Rôle |
|----------|-------|-------|------|
| 0 | from all | local | Kernel |
| 32765 | from 10.10.0.2 | 100 | pfSense WAN → internet via WiFi |
| 32766 | from all | main | Routage principal |
| 32767 | from all | default | Fallback |

### pfSense — table de routage

| Destination | Via | Interface |
|-------------|-----|-----------|
| default | 10.10.0.1 | vtnet0 (WAN) |
| 10.10.0.0/30 | — | vtnet0 (WAN) |
| 10.10.10.0/24 | — | vtnet1.10 (MGMT) |
| 10.10.20.0/24 | — | vtnet1.20 (LAN) |
| 10.10.30.0/24 | — | vtnet1.30 (SVC) |
| … | — | vtnet1.x |

### PC Admin — table de routage

| Destination | Masque | Via | Interface |
|-------------|--------|-----|-----------|
| 0.0.0.0 | 0.0.0.0 | 10.10.10.1 | Ethernet (10.10.10.7) |
| 10.10.10.0 | 255.255.255.0 | On-link | Ethernet (10.10.10.7) |

> Le PC Admin est sur VLAN MGMT via le switch. La gateway par défaut est `10.10.10.1` (pfSense vtnet1.10). Pas de routes statiques manuelles nécessaires.

---

## Règles iptables — Proxmox

### NAT POSTROUTING

| Source | Interface sortie | Action | Rôle |
|--------|-----------------|--------|------|
| 10.10.0.0/30 | wlp4s0 | MASQUERADE | NAT trafic WAN pfSense |
| 10.10.0.0/8 | wlp4s0 | MASQUERADE | NAT tout le lab |

Persistant via `post-up` dans `/etc/network/interfaces` (section `vmbr1`).

### FILTER FORWARD

| Interface entrée | Interface sortie | État | Action | Rôle |
|-----------------|-----------------|------|--------|------|
| vmbr1 | wlp4s0 | any | ACCEPT | pfSense → internet |
| wlp4s0 | vmbr1 | RELATED,ESTABLISHED | ACCEPT | Réponses internet → pfSense |

Persistant via le service `wifi-start` (`/usr/local/sbin/wifi-start`).

---

## Flux de trafic

### pfSense → Internet

```
vtnet0 (10.10.0.2)
    → vmbr1 (10.10.0.1)        [ip rule : from 10.10.0.2 → table 100]
    → wlp4s0 (172.20.10.10)    [FORWARD vmbr1 → wlp4s0 ACCEPT]
    → NAT MASQUERADE            [src remplacée par 172.20.10.10]
    → Téléphone (172.20.10.1)
    → Internet
```

### PC Admin → Proxmox UI

```
PC Admin (10.10.10.7)
    → sw-backbone-01 gi7 (untagged VLAN10)
    → sw-backbone-01 gi2 (tagué VLAN10)
    → nic0 → vmbr2 → vmbr2.10 (10.10.10.10)
    → https://10.10.10.10:8006
```

### PC Admin → pfSense UI

```
PC Admin (10.10.10.7)
    → sw-backbone-01 → vmbr2 → vtnet1.10
    → https://10.10.10.1
```

### VMs lab → Internet

```
VM (10.10.x.x)
    → vtnet1.x pfSense (gateway VLAN)
    → pfSense routing/NAT
    → vtnet0 WAN (10.10.0.2)
    → vmbr1 Proxmox → table 100 → wlp4s0
    → Téléphone → Internet
```

---

## Uplink WiFi

L'ensemble du chemin internet repose sur `wlp4s0` (partage WiFi téléphone). Cette configuration sera remplacée par :

- Connexion filaire du port WAN Proxmox directement via le switch (VLAN dédié ou port uplink)
- Box ISP en mode bridge/DMZ → pfSense récupère l'IP publique directement
- Suppression du double NAT, de la table 100 et des règles iptables MASQUERADE côté Proxmox

> Tant que l'uplink WiFi est actif : ne pas supprimer le service `wifi-start`, et ne pas retirer le `post-up` de `wlp4s0`.
