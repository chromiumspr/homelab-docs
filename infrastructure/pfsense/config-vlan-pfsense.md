# pfSense — Configuration VLANs (Router-on-a-Stick)

## Contexte

- **Hyperviseur** : Proxmox VE (`srv-proxmox-01`)
- **VM pfSense** : ID 100 (`vm-pfsense-01`)
- **Objectif** : Configurer 7 VLANs via pfSense en architecture router-on-a-stick
- **Accès temporaire** : WiFi `wlp4s0` (172.20.10.10) + hub (`vmbr0` 192.168.10.10)

---

## Architecture réseau cible

```
Proxmox vmbr2 (trunk, bridge-vlan-aware)
    └── tap100i1 → pfSense vtnet1 (trunk, pas d'IP)
                        ├── vtnet1.10  → VLAN10 MGMT  10.10.10.1/24
                        ├── vtnet1.20  → VLAN20 LAN   10.10.20.1/24
                        ├── vtnet1.30  → VLAN30 SVC   10.10.30.1/24
                        ├── vtnet1.40  → VLAN40 DMZ   10.10.40.1/24
                        ├── vtnet1.50  → VLAN50 CYB   10.10.50.1/24
                        ├── vtnet1.60  → VLAN60 FUNLAB 10.10.60.1/24
                        └── vtnet1.70  → VLAN70 BCKP  10.10.70.1/24
```

### Plan d'adressage

| VLAN | Nom    | Réseau         | Gateway     | DHCP Range                    |
|------|--------|----------------|-------------|-------------------------------|
| 10   | MGMT   | 10.10.10.0/24  | 10.10.10.1  | 10.10.10.100 → 10.10.10.200   |
| 20   | LAN    | 10.10.20.0/24  | 10.10.20.1  | 10.10.20.100 → 10.10.20.200   |
| 30   | SVC    | 10.10.30.0/24  | 10.10.30.1  | 10.10.30.100 → 10.10.30.200   |
| 40   | DMZ    | 10.10.40.0/24  | 10.10.40.1  | 10.10.40.100 → 10.10.40.200   |
| 50   | CYB    | 10.10.50.0/24  | 10.10.50.1  | 10.10.50.100 → 10.10.50.200   |
| 60   | FUNLAB | 10.10.60.0/24  | 10.10.60.1  | 10.10.60.100 → 10.10.60.200   |
| 70   | BCKP   | 10.10.70.0/24  | 10.10.70.1  | 10.10.70.100 → 10.10.70.200   |

---

## Problèmes rencontrés et résolutions

### 1. Bridge Proxmox ne laissait pas passer les VLANs

**Symptôme** : ping vers `10.10.10.1` retournait `Destination Host Unreachable`, tcpdump sur `vmbr2.10` ne capturait rien.

**Diagnostic** :
```bash
sudo bridge vlan show dev vmbr2
# Résultat : seulement VLAN 1 autorisé
```

**Fix** :
```bash
sudo bridge vlan add vid 10 dev vmbr2 self
sudo bridge vlan add vid 20 dev vmbr2 self
# ... répéter pour 30, 40, 50, 60, 70
```

---

### 2. Double tagging VLAN (invalid protocol côté pfSense)

**Symptôme** : pfSense recevait les paquets mais répondait "invalid protocol" au packet capture.

**Cause** : `vmbr2.10` côté Proxmox taggait les trames VLAN10 avant de les envoyer à pfSense. pfSense les re-taggait sur `vtnet1.10` → double tag.

**Fix** : Supprimer `vmbr2.10` et laisser pfSense gérer seul le tagging via vtnet1 trunk.

```bash
sudo ip link set vmbr2.10 down
sudo ip link delete vmbr2.10
```

Puis recréer `vmbr2.10` **après** que pfSense soit configuré en trunk, pour que Proxmox puisse joindre le VLAN10 sans interférer avec le trunk pfSense :

```bash
sudo ip link add link vmbr2 name vmbr2.10 type vlan id 10
sudo ip link set vmbr2.10 up
sudo ip addr add 10.10.10.10/24 dev vmbr2.10
```

---

### 3. Règle firewall pfSense en TCP uniquement

**Symptôme** : pfSense recevait les paquets ICMP (visible au packet capture) mais ne répondait pas.

**Cause** : La règle firewall sur l'interface MGMT était en `IPv4 TCP` uniquement, bloquant ICMP.

**Fix** : Éditer la règle → Protocol → `Any`.

---

### 4. Conflit d'IP lors de la migration LAN vers VLAN20

**Symptôme** : pfSense refusait d'assigner `10.10.20.1/24` sur `vtnet1.20` car `vtnet1` (LAN) utilisait déjà cette IP.

**Fix** :
1. Désactiver le DHCP sur LAN : **Services → DHCP Server → LAN** → décocher Enable
2. Changer l'IP de LAN temporairement vers `10.10.20.254/24`
3. Créer et configurer VLAN20 avec `10.10.20.1/24`
4. Retirer l'IP de vtnet1 LAN : IPv4 Config Type → **None**

---

## Configuration Proxmox

### /etc/network/interfaces — ajout de vmbr2.10

```
auto vmbr2
iface vmbr2 inet static
        address 10.10.20.10/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes

auto vmbr2.10
iface vmbr2.10 inet static
        address 10.10.10.10/24
        vlan-raw-device vmbr2
```

> `vmbr2.10` permet à Proxmox de joindre le VLAN MGMT (10.10.10.0/24) via pfSense.

---

## Configuration pfSense

### Étape 1 — Créer les VLANs

**Interfaces → Assignments → VLANs → Add**

Pour chaque VLAN : Parent = `vtnet1`, Tag = numéro du VLAN, Description = nom.

### Étape 2 — Assigner les interfaces

**Interfaces → Assignments** → ajouter chaque `vtnet1.xx` → crée des interfaces OPTx.

### Étape 3 — Configurer chaque interface

Pour chaque interface OPTx :
- Enable ✅
- Description : `VLANxx_NOM`
- IPv4 Config Type : Static
- IPv4 Address : `10.10.x.1/24`
- Save → Apply Changes

### Étape 4 — Retirer l'IP de vtnet1 (LAN d'origine)

**Interfaces → LAN** → IPv4 Config Type → **None** → Save → Apply

### Étape 5 — Configurer le DHCP

**Services → DHCP Server** → pour chaque interface VLAN :
- Enable ✅
- Range : `.100` → `.200`
- Gateway et DNS : laisser vide (pfSense utilise l'IP de l'interface automatiquement)
- Save

---

## État final

### Proxmox

| Interface | IP | Rôle |
|-----------|-----|------|
| vmbr0 | 192.168.10.10/24 | Gestion temporaire via hub |
| vmbr1 | 10.10.0.1/30 | WAN pfSense |
| vmbr2 | 10.10.20.10/24 | Trunk LAN vers pfSense |
| vmbr2.10 | 10.10.10.10/24 | Accès VLAN MGMT |

### pfSense

| Interface | IP | Rôle |
|-----------|-----|------|
| vtnet0 | 10.10.0.2/30 | WAN |
| vtnet1 | None | Trunk LAN |
| vtnet1.10 | 10.10.10.1/24 | MGMT |
| vtnet1.20 | 10.10.20.1/24 | LAN |
| vtnet1.30 | 10.10.30.1/24 | SVC |
| vtnet1.40 | 10.10.40.1/24 | DMZ |
| vtnet1.50 | 10.10.50.1/24 | CYB |
| vtnet1.60 | 10.10.60.1/24 | FUNLAB |
| vtnet1.70 | 10.10.70.1/24 | BCKP |

---

## TODO

- [ ] Configurer les règles firewall inter-VLANs dans pfSense
- [ ] Tester le DHCP sur chaque VLAN
- [ ] Migrer la gestion Proxmox vers VLAN MGMT (`10.10.10.10`)
- [ ] Connecter switch manageable + câble RJ45
- [ ] Supprimer `vmbr0` et débrancher le hub
- [ ] Rendre les routes Windows persistantes
- [ ] Remplacer WiFi par câble RJ45 (setup final)
