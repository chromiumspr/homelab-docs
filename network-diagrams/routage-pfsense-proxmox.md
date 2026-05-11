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
| nic0 | physique | — | Hub | Esclave de vmbr0 |
| wlp4s0 | WiFi | 172.20.10.10/28 (DHCP) | Téléphone | Internet |
| vmbr0 | bridge (nic0) | 192.168.10.10/24 | Hub → PC Admin | Gestion Proxmox |
| vmbr1 | bridge virtuel | 10.10.0.1/30 | vtnet0 pfSense | Lien WAN point-à-point |
| vmbr2 | bridge virtuel VLAN-aware | 10.10.20.10/24 | vtnet1 pfSense | LAN interne |
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
| Ethernet | 192.168.10.7/24, gw 192.168.10.1 | Hub | Gestion Proxmox |

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
