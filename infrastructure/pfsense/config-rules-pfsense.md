# pfSense — Règles Firewall


## Politique générale

- **Deny-all implicite** sur toutes les interfaces 
- Les règles s'appliquent sur l'**interface source** 
- pfSense est **stateful** — les réponses aux connexions autorisées passent automatiquement sans règle de retour explicite
- L'ordre des règles est **critique** — pfSense lis de haut en bas

---

## Aliases

### Réseaux (type : Network)

| Nom | Valeur | Description |
|-----|--------|-------------|
| `NET_MGMT` | 10.10.10.0/24 | VLAN10 MNGMT |
| `NET_LAN` | 10.10.20.0/24 | VLAN20 LAN |
| `NET_SVC` | 10.10.30.0/24 | VLAN30 SVC |
| `NET_DMZ` | 10.10.40.0/24 | VLAN40 DMZ |
| `NET_CYB` | 10.10.50.0/24 | VLAN50 CYB |
| `NET_FUNLAB` | 10.10.60.0/24 | VLAN60 FUNLAB |
| `NET_BCKP` | 10.10.70.0/24 | VLAN70 BCKP |
| `NET_PRIVATE_IPs` | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Toutes IPs privées |
| `NET_ALL_VLANS` | 10.10.10.0/24 → 10.10.70.0/24 | Tous les VLANs internes |

### Hôtes (type : Host)

| Nom | IP | Description |
|-----|----|-------------|
| `HOST_PROXMOX` | 10.10.10.10 | Proxmox VE (interface MGMT) |
| `HOST_PFSENSE_MGMT` | 10.10.10.1 | pfSense gateway MGMT |

### Ports (type : Port)

| Nom | Ports | Description |
|-----|-------|-------------|
| `PORT_WEB` | 80, 443 | HTTP/HTTPS |
| `PORT_DNS` | 53 | DNS |
| `PORT_SSH` | 22 | SSH |
| `PORT_PROXMOX` | 8006 | Interface web Proxmox |
| `PORT_PFSENSE` | 443 | Interface web pfSense |
| `PORT_WIREGUARD` | 51820 | WireGuard VPN |
| `PORT_PLEX` | 32400 | Plex Media Server |
| `PORT_BACKUP` | 8007 | Proxmox Backup Server |

---

## Règles WAN

**Firewall → Rules → WAN**

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | TCP | * | WAN address | 443 | Bloquer accès pfSense depuis Internet |
| 2 | Block | TCP | * | WAN address | 8006 | Bloquer accès Proxmox depuis Internet |
| 3 | Block | * | `NET_PRIVATE_IPs` | * | * | Anti-spoofing IP privées en entrée |
| 4 | Pass | UDP | * | WAN address | 51820 | WireGuard entrant |

---

## VLAN10 — MGMT

**Firewall → Rules → VLAN10_MGMT**

**Politique** : Admin only — accès strict aux interfaces d'administration uniquement. 
| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Pass | TCP | `NET_MGMT` | `HOST_PROXMOX` | 8006 | Accès interface web Proxmox |
| 2 | Pass | TCP | `NET_MGMT` | `HOST_PFSENSE_MGMT` | 443 | Accès interface web pfSense |
| 3 | Pass | TCP | `NET_MGMT` | `NET_MGMT` | 22 | SSH intra-MGMT |
| 4 | Pass | TCP/UDP | `NET_MGMT` | * | 53 | DNS sortant |
| 5 | Block | * | `NET_MGMT` | * | * | Deny tout le reste |

> **Évolution** : Ajouter une règle Pass par service au fur et à mesure des déploiements (Portainer, etc.) en ciblant l'host spécifique et le port approprié.

---

## VLAN20 — LAN

**Firewall → Rules → VLAN20_LAN**

**Politique** : Accès Internet autorisé. Accès aux services internes via NPM proxy uniquement (port 443). Isolation totale des VLANs sensibles.

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | * | `NET_LAN` | `NET_MGMT` | * | Deny LAN → MGMT |
| 2 | Block | * | `NET_LAN` | `NET_DMZ` | * | Deny LAN → DMZ |
| 3 | Block | * | `NET_LAN` | `NET_CYB` | * | Deny LAN → CYB |
| 4 | Block | * | `NET_LAN` | `NET_FUNLAB` | * | Deny LAN → FUNLAB |
| 5 | Block | * | `NET_LAN` | `NET_BCKP` | * | Deny LAN → BCKP |
| 6 | Pass | TCP/UDP | `NET_LAN` | `HOST_PIHOLE` | 53 | DNS → PiHole |
| 7 | Pass | TCP | `NET_LAN` | `HOST_NPM_SVC` | 443 | Accès services internes via NPM proxy |
| 8 | Pass | TCP | `NET_LAN` | * | 80, 443 | Accès Internet |
| 9 | Block | * | `NET_LAN` | * | * | Deny tout le reste |

> Les machines LAN accèdent aux services SVC **uniquement via NPM** . Tout accès direct à une IP SVC sur un port non-443 est bloqué par la règle 9.

---

## VLAN30 — SVC

**Firewall → Rules → VLAN30_SVC**

**Politique** : Les services peuvent accéder à Internet pour les mises à jour. Aucune initiation de connexion vers les autres VLANs. 

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | * | `NET_SVC` | `NET_MGMT` | * | Deny SVC → MGMT |
| 2 | Block | * | `NET_SVC` | `NET_LAN` | * | Deny SVC → LAN |
| 3 | Block | * | `NET_SVC` | `NET_DMZ` | * | Deny SVC → DMZ |
| 4 | Block | * | `NET_SVC` | `NET_CYB` | * | Deny SVC → CYB |
| 5 | Block | * | `NET_SVC` | `NET_FUNLAB` | * | Deny SVC → FUNLAB |
| 6 | Pass | TCP/UDP | `NET_SVC` | * | 53 | DNS sortant (PiHole upstream) |
| 7 | Pass | TCP | `NET_SVC` | * | 80, 443 | Accès Internet (mises à jour, apt, Docker) |
| 8 | Block | * | `NET_SVC` | * | * | Deny tout le reste |

---

## VLAN40 — DMZ

**Firewall → Rules → VLAN40_DMZ**

**Politique** : Deny RFC1918 total — la DMZ ne peut jamais atteindre le réseau privé. Accès Internet uniquement pour les mises à jour. L'accès entrant depuis Internet (Plex) est géré par une règle WAN + NAT Port Forward.

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | * | `NET_DMZ` | `NET_PRIVATE_IPs` | * | Deny DMZ → tout le réseau privé |
| 2 | Pass | TCP/UDP | `NET_DMZ` | * | 53 | DNS sortant (resolver externe) |
| 3 | Pass | TCP | `NET_DMZ` | * | 80, 443 | Accès Internet |
| 4 | Block | * | `NET_DMZ` | * | * | Deny tout le reste |

> **Important** : La règle 1 doit être en **première position** — elle bloque également l'accès à PiHole (SVC). Le DNS DMZ utilise donc un resolver externe (1.1.1.1 ou 8.8.8.8).

---

## VLAN50 — CYB

**Firewall → Rules → VLAN50_CYB**

**Politique** : Internet only. Isolation totale du réseau interne. Prévu pour les labs de sécurité avec machines potentiellement compromises.

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | * | `NET_CYB` | `NET_ALL_VLANS` | * | Deny CYB → tous les VLANs internes |
| 2 | Pass | TCP/UDP | `NET_CYB` | * | 53 | DNS sortant |
| 3 | Pass | TCP | `NET_CYB` | * | 80, 443 | Accès Internet |
| 4 | Block | * | `NET_CYB` | * | * | Deny tout le reste |

> **Option** : Pour des labs offensifs (scans, exploits), désactiver les règles 2 et 3 via le bouton **Toggle** pour passer en mode fully isolated temporairement.

---

## VLAN60 — FUNLAB

**Firewall → Rules → VLAN60_FUNLAB**

**Politique** : Héberge la stack *arr + qBittorrent + Homarr. Accès Internet autorisé. Accès aux services internes via NPM uniquement. Isolation des VLANs sensibles.

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Block | * | `NET_FUNLAB` | `NET_MGMT` | * | Deny FUNLAB → MGMT |
| 2 | Block | * | `NET_FUNLAB` | `NET_CYB` | * | Deny FUNLAB → CYB |
| 3 | Block | * | `NET_FUNLAB` | `NET_BCKP` | * | Deny FUNLAB → BCKP |
| 4 | Pass | TCP/UDP | `NET_FUNLAB` | `HOST_PIHOLE` | 53 | DNS → PiHole |
| 5 | Pass | TCP | `NET_FUNLAB` | `HOST_NPM_SVC` | 443 | Accès services internes via NPM proxy |
| 6 | Pass | TCP | `NET_FUNLAB` | * | 80, 443 | Accès Internet (APIs *arr, mises à jour) |
| 7 | Pass | TCP/UDP | `NET_FUNLAB` | * | 6881 | qBittorrent (port torrent) |
| 8 | Block | * | `NET_FUNLAB` | * | * | Deny tout le reste |

> **Recommandation** : Faire passer qBittorrent via un container VPN **Gluetun** pour isoler le trafic torrent. Le port 6881 est configurable dans les settings qBittorrent — adapter la règle 7 en conséquence.

---

## VLAN70 — BCKP

**Firewall → Rules → VLAN70_BCKP**

**Politique** : Pull only — PBS initie toutes les sauvegardes/restaurations. Personne ne peut pousser vers BCKP. Protection ransomware assurée par l'absence de règles autorisant les autres VLANs à initier des connexions vers NET_BCKP.

| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Pass | TCP | `NET_BCKP` | `NET_ALL_VLANS` | 8007 | PBS pull/restore toutes VMs |
| 2 | Pass | TCP/UDP | `NET_BCKP` | * | 53 | DNS sortant |
| 3 | Block | * | `NET_BCKP` | * | * | Deny tout le reste |

> Les mises à jour PBS sont gérées manuellement via SSH depuis MGMT (`apt upgrade`) — pas d'accès Internet direct depuis BCKP.

---

## Matrice inter-VLAN (synthèse)

| Source ↓ \ Dest → | MGMT | LAN | SVC | DMZ | CYB | FUNLAB | BCKP | Internet |
|---|---|---|---|---|---|---|---|---|
| **MGMT** | SSH | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| **LAN** | ✗ | ✓ | NPM:443 | ✗ | ✗ | ✗ | ✗ | ✓ |
| **SVC** | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **DMZ** | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓ |
| **CYB** | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ |
| **FUNLAB** | ✗ | ✗ | NPM:443 | ✗ | ✗ | ✓ | ✗ | ✓ |
| **BCKP** | PBS:8007 | PBS:8007 | PBS:8007 | PBS:8007 | ✗ | PBS:8007 | ✓ | ✗ |

> ✓ = autorisé (intra-VLAN) | ✗ = bloqué | NPM:443 = via proxy uniquement | PBS:8007 = sauvegarde uniquement

---

## TODO — Règles à ajouter lors des déploiements

- [ ] **Plex** : Règle WAN `Pass TCP/UDP * → WAN:32400` + NAT Port Forward vers `HOST_NPM_DMZ`
- [ ] **Portainer** : Règle MGMT `Pass TCP NET_MGMT → HOST_PORTAINER : 9000/9443`
- [ ] **WireGuard server** : Définir l'interface WireGuard dans pfSense et les règles associées
- [ ] Mettre à jour les aliases `HOST_PIHOLE`, `HOST_NPM_SVC`, `HOST_NPM_DMZ`, `HOST_PBS` avec les IPs réelles après déploiement des VMs
