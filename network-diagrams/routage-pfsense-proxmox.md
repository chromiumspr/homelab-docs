# Routage complet : Interfaces, routes et règles

## Inventaire des machines

| Machine | Rôle |
|---------|------|
| srv-proxmox-01 | Hyperviseur Proxmox, routeur NAT |
| vm-pfsense-01 | Routeur/Firewall pfSense (VM ID 100) |
| PC Admin | Poste d'administration Windows |
| Téléphone | Partage de connexion WiFi (internet) |

---

## Interfaces réseau

### srv-proxmox-01

| Interface | Type | IP | Connecté à | Rôle |
|-----------|------|----|-----------|------|
| nic0 | physique | — | sw-backbone-01 | Esclave de vmbr2.10 |
| wlp4s0 | WiFi | 172.20.10.10/28 (DHCP) | Téléphone | Internet |
| vmbr1 | bridge virtuel | 10.10.0.1/30 | vtnet0 pfSense | Lien WAN point-à-point |
| vmbr2 | bridge virtuel VLAN-aware | 10.10.20.10/24 | vtnet1 pfSense | LAN interne |
| vmbr2.10 | — | 10.10.10.10./24 | — | Gestion Proxmox Web :8006 |
| tap100i0 | tap VM | — | vmbr1 | Interface réseau VM pfSense (WAN) |
| tap100i1 | tap VM | — | vmbr2 | Interface réseau VM pfSense (LAN) |

### vm-pfsense-01

| Interface | Type | IP | Connecté à | Rôle |
|-----------|------|----|-----------|------|
| vtnet0 | virtio | 10.10.0.2/30, gw 10.10.0.1 | vmbr1 Proxmox | WAN |
| vtnet1 | virtio | 10.10.20.1/24 | vmbr2 Proxmox | LAN |

### PC Admin

| Interface | IP | Connecté à | Rôle |
|-----------|----|-----------|------|
| Ethernet | 10.10.10.7/24, gw 192.168.10.1 | sw-backbone-01 | Gestion Proxmox |

---

## Interconnexions

```
[Téléphone 172.20.10.1]
        │ WiFi 802.11
        │
[wlp4s0 172.20.10.10] ← Proxmox
        │ NAT MASQUERADE
        │ ip rule table 100
        │
[vmbr1 10.10.0.1/30] ←────────────────────────── [vtnet0 10.10.0.2/30]
                                                           │
                                                    pfSense VM
                                                           │
[vmbr2 10.10.20.10/24] ←──────────────────────── [vtnet1 10.10.20.1/24]
        │
        └── VMs du lab (10.10.x.x)

[vmbr0 192.168.10.10/24]
        │
       Hub
        │
[PC Admin 192.168.10.7/24]
```

---

## Tables de routage

### Proxmox — table main

```
ip route show
```

| Destination | Via | Interface | Source |
|------------|-----|-----------|--------|
| default | 192.168.10.1 | vmbr0 | — |
| 10.10.0.0/30 | — | vmbr1 | 10.10.0.1 |
| 10.10.20.0/24 | — | vmbr2 | 10.10.20.10 |
| 172.20.10.0/28 | — | wlp4s0 | 172.20.10.10 |
| 192.168.10.0/24 | — | vmbr0 | 192.168.10.10 |

### Proxmox — table 100

```
ip route show table 100
```

| Destination | Via | Interface |
|------------|-----|-----------|
| default | 172.20.10.1 | wlp4s0 |

Utilisée uniquement pour les paquets dont la source est `10.10.0.2` (pfSense WAN). Tout sort par le WiFi vers internet.

### Proxmox — ip rule

```
ip rule show
```

| Priorité | Règle | Table | Rôle |
|----------|-------|-------|------|
| 0 | from all | local | Kernel |
| 32765 | from 10.10.0.2 | 100 | pfSense WAN → internet via WiFi |
| 32766 | from all | main | Routage principal |
| 32767 | from all | default | Fallback |

### pfSense — table de routage

```
netstat -rn (depuis pfSense)
```

| Destination | Via | Interface |
|------------|-----|-----------|
| default | 10.10.0.1 | vtnet0 (WAN) |
| 10.10.0.0/30 | — | vtnet0 (WAN) |
| 10.10.20.0/24 | — | vtnet1 (LAN) |

### PC Admin — table de routage Windows

```
route print
```

| Destination | Masque | Via | Interface |
|------------|--------|-----|-----------|
| 0.0.0.0 | 0.0.0.0 | 192.168.10.1 | Ethernet (192.168.10.7) |
| 10.0.0.0 | 255.0.0.0 | 192.168.10.10 | Ethernet (192.168.10.7) |
| 10.10.20.0 | 255.255.255.0 | 192.168.10.10 | Ethernet (192.168.10.7) |
| 192.168.10.0 | 255.255.255.0 | On-link | Ethernet (192.168.10.7) |

> ⚠️ Les routes Windows `10.0.0.0/8` et `10.10.20.0/24` sont temporaires — elles disparaissent au reboot.

---

## Règles iptables — Proxmox

### NAT POSTROUTING

```
iptables -t nat -L POSTROUTING -v
```

| Source | Interface sortie | Action | Rôle |
|--------|-----------------|--------|------|
| 10.10.0.0/30 | wlp4s0 | MASQUERADE | NAT trafic WAN pfSense |
| 10.10.0.0/8 | wlp4s0 | MASQUERADE | NAT tout le lab |

Remplace l'IP source par `172.20.10.10` avant d'envoyer sur le WiFi. Le téléphone ne voit que cette IP.

Persistant via `post-up` dans `/etc/network/interfaces` (section `vmbr1`).

### FILTER FORWARD

```
iptables -L FORWARD -v
```

| Interface entrée | Interface sortie | État | Action | Rôle |
|-----------------|-----------------|------|--------|------|
| vmbr1 | wlp4s0 | any | ACCEPT | pfSense → internet |
| wlp4s0 | vmbr1 | RELATED,ESTABLISHED | ACCEPT | Réponses internet → pfSense |

Persistant via script `wifi-start`.

---

## Flux de trafic

### pfSense → internet

```
vtnet0 (10.10.0.2)
    → vmbr1 (10.10.0.1)         [ip rule : from 10.10.0.2 → table 100]
    → wlp4s0 (172.20.10.10)     [FORWARD vmbr1 → wlp4s0 ACCEPT]
    → NAT MASQUERADE             [src remplacée par 172.20.10.10]
    → Téléphone (172.20.10.1)
    → Internet
```

### PC Admin → interface web pfSense

```
PC Admin (192.168.10.7)
    → route : 10.10.20.0/24 via 192.168.10.10
    → vmbr0 (192.168.10.10) Proxmox
    → vmbr2 (10.10.20.10)
    → vtnet1 pfSense (10.10.20.1)
    → http://10.10.20.1
```

### VMs du lab → internet (futur)

```
VM (10.10.2x.x)
    → vtnet1 pfSense LAN
    → pfSense routing/NAT
    → vtnet0 WAN (10.10.0.2)
    → vmbr1 Proxmox
    → table 100 → wlp4s0
    → Téléphone → Internet
```

---

## Persistance des règles

| Règle | Persistant via | Survivre au reboot |
|-------|---------------|-------------------|
| ip rule from 10.10.0.2 table 100 | wifi-start script | ✅ |
| ip route table 100 default | wifi-start script | ✅ |
| iptables NAT MASQUERADE | /etc/network/interfaces post-up vmbr1 | ✅ |
| iptables FORWARD | wifi-start script | ✅ |
| ip forward (sysctl) | /etc/network/interfaces post-up vmbr1 | ✅ |
| Routes Windows 10.x.x.x | route add manuel | ❌ à refaire au reboot |





# Routage complet : Interfaces, routes et règles — v4.0

> **Mise à jour :** Architecture "Philosophie B" avec switch manageable Cisco SG350-28  
> **Remplace :** Version antérieure basée sur hub + `vmbr0` actif + LAN flat

---

## Inventaire des machines

| Machine | Rôle |
|---------|------|
| srv-proxmox-01 | Hyperviseur Proxmox, routeur NAT temporaire |
| vm-pfsense-01 | Routeur/Firewall pfSense (VM ID 100) |
| sw-backbone-01 | Switch manageable Cisco SG350-28 (`10.10.10.2/24`) |
| PC Admin | Poste d'administration (`10.10.10.7/24`) |
| Téléphone | Partage de connexion WiFi (internet temporaire) |

---

## Interfaces réseau

### srv-proxmox-01

| Interface | Type | IP | Connecté à | Rôle |
|-----------|------|----|------------|------|
| nic0 | physique | — | sw-backbone-01 gi2 | Esclave de vmbr2 (trunk VLAN-aware) |
| wlp4s0 | WiFi | 172.20.10.10/28 (DHCP) | Téléphone | Internet temporaire |
| vmbr0 | bridge (`bridge-ports none`) | 192.168.10.10/24 | — | Inactif — fallback uniquement |
| vmbr1 | bridge virtuel point-à-point | 10.10.0.1/30 | vtnet0 pfSense | Lien WAN |
| vmbr2 | bridge virtuel VLAN-aware | `inet manual` (pas d'IP) | nic0 → switch gi2 | Trunk multi-VLAN |
| vmbr2.10 | sous-interface VLAN 10 | 10.10.10.10/24 | — | Accès management Proxmox |
| tap100i0 | tap VM | — | vmbr1 | Interface WAN VM pfSense |
| tap100i1 | tap VM | — | vmbr2 | Interface LAN/trunk VM pfSense |

> ⚠️ `vmbr2` doit rester `inet manual` (sans IP directe). L'accès management Proxmox passe **uniquement** par `vmbr2.10`.  
> ⚠️ `vmbr0` est conservé en structure mais inactif (`bridge-ports none`). Ne pas y remettre de gateway.

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

## Persistance des règles

| Règle | Persistant via | Survie au reboot |
|-------|---------------|-----------------|
| ip rule from 10.10.0.2 table 100 | `/usr/local/sbin/wifi-start` (systemd service) | ✅ |
| ip route table 100 default | `/usr/local/sbin/wifi-start` | ✅ |
| iptables NAT MASQUERADE | `post-up` vmbr1 dans `/etc/network/interfaces` | ✅ |
| iptables FORWARD | `/usr/local/sbin/wifi-start` | ✅ |
| ip_forward sysctl | `post-up` vmbr1 dans `/etc/network/interfaces` | ✅ |
| vmbr2.10 (10.10.10.10/24) | `/etc/network/interfaces` | ✅ |
| default route via wlp4s0 | `post-up` wlp4s0 dans `/etc/network/interfaces` | ✅ |

---

## Ce qui est temporaire (uplink WiFi)

L'ensemble du chemin internet repose sur `wlp4s0` (partage WiFi téléphone). Cette configuration sera remplacée par :

- Connexion filaire du port WAN Proxmox directement via le switch (VLAN dédié ou port uplink)
- Box ISP en mode bridge/DMZ → pfSense récupère l'IP publique directement
- Suppression du double NAT, de la table 100 et des règles iptables MASQUERADE côté Proxmox

> Tant que l'uplink WiFi est actif : ne pas supprimer le service `wifi-start`, et ne pas retirer le `post-up` de `wlp4s0`.
