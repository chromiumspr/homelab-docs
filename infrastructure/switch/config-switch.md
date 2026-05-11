# Switch Manageable SG350

**Date :** 27/04/2026  

## Objectif

Remplacer le hub temporaire par le switch manageable Cisco SG350-28 afin de :
- Accéder à Proxmox via VLAN MGMT (`10.10.10.10`) au lieu de l'ancien accès hub (`192.168.10.10`)
- Mettre en place un trunk 802.1Q propre entre Proxmox et le switch
- Poser les bases pour le déploiement futur de tous les VLANs (10-70)

---

## Topologie cible

```
PC (10.10.10.7) ──── gi7 [untagged VLAN10]
                          sw-backbone-01 (SG350-28)
Proxmox (nic0) ──── gi2 [trunk tagué VLANs 10-70, PVID=10]
```

---

## 1. Configuration du switch SG350

### Reset usine

Bouton reset maintenu ~10 secondes (LED System clignote pendant le boot, comportement normal).

Credentials usine : `cisco` / `cisco`

### IP de management

```
configure terminal
interface vlan 10
ip address 10.10.10.2 255.255.255.0
no shutdown
exit
ip default-gateway 10.10.10.1
no ip http server
ip http secure-server
```

### Hostname

```
configure terminal
hostname sw-backbone-01
end
```

### Création des VLANs

```
configure terminal
vlan 10,20,30,40,50,60,70
end
```

### Configuration des ports

**gi2 → Proxmox (trunk)**

```
configure terminal
interface gi2
switchport mode trunk
switchport trunk native vlan 10
exit
end
```

**gi7 → PC (access VLAN10)**

```
configure terminal
interface gi7
switchport access vlan 10
exit
end
```

> **Note :** Le `native vlan 10` sur `gi2` est indispensable — sans lui, le trafic non tagué de Proxmox arrivait sur VLAN1 et était filtré.

### Résultat `show vlan`

| VLAN | Tagged Ports | Untagged Ports |
|------|-------------|----------------|
| 1    |             | gi1-6, gi8-28  |
| 10   | gi2         | gi7            |
| 20   | gi2         |                |
| 30   | gi2         |                |
| 40   | gi2         |                |
| 50   | gi2         |                |
| 60   | gi2         |                |
| 70   | gi2         |                |

### Sauvegarde

```
copy running-config startup-config
```

---

## 2. Configuration Proxmox

### Modification `/etc/network/interfaces`

Déplacement de `nic0` de `vmbr0` vers `vmbr2` :

```
# Avant
auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    bridge-ports nic0        ← nic0 était ici
    bridge-stp off
    bridge-fd 0

auto vmbr2
iface vmbr2 inet static
    address 10.10.20.10/24
    bridge-ports none        ← pas de port physique
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

```
# Après
auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    bridge-ports none        ← hub retiré
    bridge-stp off
    bridge-fd 0

auto vmbr2
iface vmbr2 inet static
    address 10.10.20.10/24
    bridge-ports nic0        ← nic0 déplacé ici
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

Application :

```bash
sudo ifreload -a
```

### Correction VLAN filtering sur nic0

Par défaut, `nic0` n'avait que VLAN1 en PVID. Les trames taggées VLAN10 arrivant du switch étaient filtrées par le bridge et ne remontaient pas sur `vmbr2.10`.

Correction :

```bash
sudo bridge vlan add dev nic0 vid 10 pvid untagged
sudo bridge vlan del dev nic0 vid 1
```

Vérification :

```bash
bridge vlan show dev nic0
# Résultat attendu :
# nic0   10   PVID Egress Untagged
```

---

## 3. Mise à jour `/etc/hosts`

```
127.0.0.1   localhost
10.10.10.10 srv-proxmox-01
```

> `srv-proxmox-01` ne doit pas apparaître sur la ligne `127.0.0.1`.

Vérification :

```bash
getent hosts srv-proxmox-01
# 10.10.10.10   srv-proxmox-01
```

---

## 4. Routes Windows persistantes

Route ajoutée sur le PC (interface 13, carte Ethernet physique) :

```cmd
route add 10.0.0.0 mask 255.0.0.0 192.168.10.10 if 13 -p
```

> Cette route passait encore par le hub (`192.168.10.10`) au moment de la config. À mettre à jour lors de la suppression définitive de `vmbr0`.

---

## Problèmes rencontrés

| Problème | Cause | Solution |
|----------|-------|----------|
| Ping PC → Proxmox échoue après branchement switch | `nic0` non membre de `vmbr2` | Déplacer `nic0` dans `bridge-ports` de `vmbr2` |
| Trafic arrivait sur VLAN1 au lieu de VLAN10 | Pas de `native vlan 10` sur `gi2` | `switchport trunk native vlan 10` sur gi2 |
| Trames VLAN10 filtrées par le bridge | `nic0` avait VLAN1 comme PVID | `bridge vlan add dev nic0 vid 10 pvid untagged` |

---

## État final

| Élément | IP | Statut |
|---------|----|--------|
| Proxmox WebUI | `10.10.10.10:8006` | ✅ Accessible |
| Switch management | `10.10.10.2` | ✅ Configuré |
| PC | `10.10.10.7` | ✅ Connecté VLAN MGMT |
| Hub (`vmbr0`) | `192.168.10.10` | 🟡 Inactif (gardé en secours) |

---

