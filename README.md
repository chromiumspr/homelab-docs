# homelab-docs

# 🏠 Homelab Docs

Documentation de mon homelab auto-hébergé basé sur Proxmox VE.

## Stack

- **Hyperviseur** : Proxmox VE (`srv-proxmox-01`)
- **Firewall/Router** : pfSense
- **Switch** : Cisco SG350-28
- **Services** : Docker, PiHole, NPM, WireGuard, Plex, *arr stack

## Architecture réseau

| VLAN | Nom | Subnet | Rôle |
|---|---|---|---|
| 10 | MGMT | 10.10.10.0/24 | Proxmox, pfSense, switch |
| 20 | LAN | 10.10.20.0/24 | Clients |
| 30 | SVC | 10.10.30.0/24 | PiHole, NPM interne |
| 40 | DMZ | 10.10.40.0/24 | NPM externe, Plex |
| 50 | CYB | 10.10.50.0/24 | Pentest/CTF |
| 60 | FUNLAB | 10.10.60.0/24 | *arr, qBittorrent |
| 70 | BCKP | 10.10.70.0/24 | PBS, NAS |

## 📁 Structure
homelab-docs/
├── infrastructure/
│   ├── proxmox/
│   ├── pfsense/
│   └── switch/
├── services/
│   └── docker/
└── network-diagrams/
└── homelab-architecture.html

chromiumspr.github.io avant /homelab-docs pour visualiser le html  
```

chromiumspr.github.io avant /homelab-docs pour visualiser le html 
📊 Diagramme
👉 Voir l'architecture réseau

chromiumspr.github.io avant /homelab-docs pour visualiser le html
