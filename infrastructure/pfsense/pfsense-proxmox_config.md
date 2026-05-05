# pfSense sur Proxmox — Setup temporaire WiFi

## Contexte

Configuration d'une VM pfSense sur Proxmox dans un setup **temporaire** où internet arrive via un partage de connexion WiFi (téléphone). Le serveur dispose d'un seul port RJ45 utilisé pour la gestion, et d'une carte WiFi pour internet.

### Topologie physique

```
Téléphone (172.20.10.1)
    └── WiFi → wlp4s0 (172.20.10.10) [internet]

Hub
    └── RJ45 → nic0 → vmbr0 (192.168.10.10) [gestion Proxmox]
                          └── PC Admin (192.168.10.7)
```

---

## Plan d'adressage

| VLAN | Nom    | Subnet         | Gateway    |
|------|--------|----------------|------------|
| 10   | MGMT   | 10.10.10.0/24  | 10.10.10.1 |
| 20   | LAN    | 10.10.20.0/24  | 10.10.20.1 |
| 30   | SVC    | 10.10.30.0/24  | 10.10.30.1 |
| 40   | DMZ    | 10.10.40.0/24  | 10.10.40.1 |
| 50   | CYB    | 10.10.50.0/24  | 10.10.50.1 |
| 60   | FUNLAB | 10.10.60.0/24  | 10.10.60.1 |
| 70   | BCKP   | 10.10.70.0/24  | 10.10.70.1 |

Lien WAN point-à-point (Proxmox ↔ pfSense) : `10.10.0.0/30`

| Hôte | IP |
|------|----|
| Proxmox (vmbr1) | 10.10.0.1 |
| pfSense (vtnet0 WAN) | 10.10.0.2 |

---

## Architecture réseau

```
Téléphone (172.20.10.1)
    │ WiFi
    wlp4s0 (172.20.10.10)
    │ NAT MASQUERADE + table 100
    vmbr1 (10.10.0.1/30) ──── vtnet0 WAN (10.10.0.2) ─┐
                                                        pfSense VM
    vmbr2 (10.10.20.10/24) ── vtnet1 LAN (10.10.20.1) ─┘
    │ VLAN-aware
    └── VMs du lab (10.10.x.x)

    vmbr0 (192.168.10.10/24)
    │
    Hub → PC Admin (192.168.10.7)
```

---

## Étape 1 — Fichier `/etc/network/interfaces`

```
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.10.10/24
        gateway 192.168.10.1
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0

auto vmbr1
iface vmbr1 inet static
        address 10.10.0.1/30
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.10.0.0/30' -o wlp4s0 -j MASQUERADE
        post-up iptables -t nat -A POSTROUTING -s '10.10.0.0/8' -o wlp4s0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.0.0/30' -o wlp4s0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.0.0/8' -o wlp4s0 -j MASQUERADE

auto vmbr2
iface vmbr2 inet static
        address 10.10.20.10/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes

iface wlp4s0 inet dhcp
        wpa-conf /etc/wpa_supplicant.conf

iface nic1 inet manual

source /etc/network/interfaces.d/*
```

### Rôle des bridges

| Bridge | IP | Rôle |
|--------|----|------|
| vmbr0 | 192.168.10.10/24 | Gestion Proxmox via hub |
| vmbr1 | 10.10.0.1/30 | Lien WAN point-à-point vers pfSense |
| vmbr2 | 10.10.20.10/24 | LAN interne VLAN-aware |

---

## Étape 2 — Script WiFi au démarrage

Le WiFi ne monte pas automatiquement via `ifupdown`. Un service systemd dédié gère le démarrage et configure les règles de routage une fois l'IP obtenue.

### Script `/usr/local/sbin/wifi-start`

```bash
#!/bin/bash
ip link set wlp4s0 up
wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf
dhclient wlp4s0

# pour laisser le temps à l'interface de prendre une ip
sleep 7

# nettoyage de l'existant
ip rule del from 10.10.0.0/30 table 100 2>/dev/null
ip rule del from 10.10.0.2 table 100 2>/dev/null
ip route flush table 100 2>/dev/null

# Routing
ip rule add from 10.10.0.2 table 100
ip route add default via 172.20.10.1 dev wlp4s0 table 100

# FORWARD
iptables -F FORWARD
iptables -A FORWARD -i vmbr1 -o wlp4s0 -j ACCEPT
iptables -A FORWARD -i wlp4s0 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Service `/etc/systemd/system/wifi-start.service`

```ini
[Unit]
Description=Démarrage WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/wifi-start
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Activation :

```bash
systemctl enable wifi-start
```

---

## Étape 3 — Tables de routage et règles

### `ip rule show`

| Priorité | Règle | Table | Rôle |
|----------|-------|-------|------|
| 0 | from all | local | Géré par le kernel |
| 32765 | from 10.10.0.2 | 100 | Trafic WAN pfSense → internet |
| 32766 | from all | main | Routage principal |
| 32767 | from all | default | Fallback |

### Table main (`ip route show`)

| Destination | Via | Interface | Source |
|------------|-----|-----------|--------|
| default | 192.168.10.1 | vmbr0 | — |
| 10.10.0.0/30 | — | vmbr1 | 10.10.0.1 |
| 10.10.20.0/24 | — | vmbr2 | 10.10.20.10 |
| 172.20.10.0/28 | — | wlp4s0 | 172.20.10.10 |
| 192.168.10.0/24 | — | vmbr0 | 192.168.10.10 |

### Table 100 (`ip route show table 100`)

| Destination | Via | Interface |
|------------|-----|-----------|
| default | 172.20.10.1 | wlp4s0 |

Tout paquet dont la source est `10.10.0.2` (pfSense WAN) sort par le WiFi vers internet.

---

## Étape 4 — Règles iptables

### NAT POSTROUTING (persistant via `vmbr1` post-up)

```bash
iptables -t nat -A POSTROUTING -s '10.10.0.0/30' -o wlp4s0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s '10.10.0.0/8'  -o wlp4s0 -j MASQUERADE
```

Remplace l'IP source par `172.20.10.10` avant d'envoyer sur le WiFi. Le téléphone ne voit que l'IP WiFi de Proxmox.

### FORWARD (persistant via `wifi-start`)

```bash
iptables -A FORWARD -i vmbr1 -o wlp4s0 -j ACCEPT
iptables -A FORWARD -i wlp4s0 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

- Règle 1 : autorise le trafic de pfSense vers internet
- Règle 2 : autorise les réponses d'internet vers pfSense (connexions établies uniquement)

---

## Étape 5 — Création de la VM pfSense

### Paramètres VM (ID 100)

| Paramètre | Valeur |
|-----------|--------|
| Nom | vm-pfsense-01 |
| OS | pfSense CE 2.7.2 (FreeBSD) |
| Disque | 8GB sur local-lvm (NVMe) |
| CPU | 2 vCPU |
| RAM | 2048 MB |
| net0 | virtio, bridge=vmbr1, firewall=**non** |
| net1 | virtio, bridge=vmbr2, firewall=**non** |
| onboot | yes |

> ⚠️ Le firewall Proxmox doit être **désactivé** sur les deux interfaces réseau. Sinon Proxmox crée des bridges `fwbr` intermédiaires non connectés aux bridges principaux, cassant toute connectivité.

### Hookscript — attachement des tap au démarrage

Fichier `/var/lib/vz/snippets/pfsense-hookscript.pl` :

```perl
#!/usr/bin/perl

my $vmid = shift;
my $phase = shift;

if ($phase eq 'post-start') {
    system("brctl addif vmbr1 tap${vmid}i0");
    system("brctl addif vmbr2 tap${vmid}i1");
}
```

```bash
chmod +x /var/lib/vz/snippets/pfsense-hookscript.pl
qm set 100 --hookscript local:snippets/pfsense-hookscript.pl
```

---

## Étape 6 — Configuration pfSense

### Interfaces

| Interface pfSense | IP | Rôle |
|------------------|----|------|
| vtnet0 (WAN) | 10.10.0.2/30, gw 10.10.0.1 | Lien vers Proxmox/internet |
| vtnet1 (LAN) | 10.10.20.1/24 | LAN lab |

### DHCP LAN

- Plage : `10.10.20.100` → `10.10.20.200`

### Accès interface web pfSense

Depuis Proxmox :

```bash
curl -k http://10.10.20.1
```

Depuis PC Admin (Windows) — ajouter une route statique :

```cmd
route add 10.10.20.0 mask 255.255.255.0 192.168.10.10 if <n>
```

Puis ouvrir `http://10.10.20.1` dans un navigateur.

Identifiants par défaut : `admin` / `pfsense`

---

## Points en suspens (TODO)

- [ ] Configurer les VLANs dans pfSense (MGMT, LAN, SVC, DMZ, CYB, FUNLAB, BCKP)
- [ ] Migrer l'IP de `vmbr2` vers le VLAN MGMT (`10.10.10.10`)
- [ ] Migrer l'IP de gestion Proxmox vers le VLAN MGMT
- [ ] Autoriser le ping ICMP WAN dans pfSense (règle firewall)
- [ ] Supprimer `vmbr0` et le hub une fois le VLAN MGMT opérationnel
- [ ] Rendre les routes Windows persistantes (regedit ou script)
- [ ] Remplacer le WiFi par le câble RJ45 — setup final

---

## Problèmes rencontrés et solutions

### 1. Bridge sur interface WiFi impossible
Le protocole 802.11 interdit le bridging direct. Solution : NAT + policy routing côté Proxmox pour faire transiter le trafic internet via `wlp4s0`.

### 2. Firewall Proxmox casse la connectivité VM
Quand `firewall=1` sur une interface réseau de VM, Proxmox insère un bridge `fwbr` intermédiaire non connecté au bridge principal. Solution : désactiver le firewall Proxmox sur les deux interfaces réseau de la VM pfSense.

### 3. Tap non attachés aux bridges au boot
Les interfaces tap de la VM ne s'attachaient pas automatiquement à `vmbr1`/`vmbr2`. Solution : hookscript Proxmox qui exécute `brctl addif` en `post-start`.

### 4. WiFi ne monte pas au boot
`wlp4s0` ne montait pas automatiquement via `ifupdown`. Solution : service systemd `wifi-start` avec `wpa_supplicant` + `dhclient` explicites, et `sleep 7` pour laisser le temps d'obtenir une IP avant d'appliquer les règles de routage.

### 5. Table 100 vide au reboot
Les routes de la table 100 disparaissaient car créées avant que `wlp4s0` soit up. Solution : création dans le script `wifi-start` après `dhclient` et `sleep`.

### 6. Routing asymétrique — règle trop large
La règle `from 10.10.0.0/30 table 100` interceptait aussi les paquets de Proxmox (`10.10.0.1`) et les envoyait vers le WiFi au lieu de `vmbr1`. Solution : règle plus précise `from 10.10.0.2 table 100` (uniquement l'IP WAN de pfSense).

### 7. pfSense bloque ICMP sur WAN
Comportement par défaut de pfSense — le firewall bloque les pings entrants sur le WAN. À corriger via une règle firewall dans l'interface web pfSense.
