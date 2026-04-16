# Documentation Proxmox Homelab — srv-proxmox-01

## Matériel

| Composant | Détail |
|-----------|--------|
| NVMe | Micron 2200S 512GB |
| SATA SSD x3 | Intel SSDSC2KF256G8 256GB chacun |

---

## 1. Installation de Proxmox

### Prérequis BIOS
- Le NVMe n'était pas détecté car le BIOS était en mode **Intel RST/VMD**
- Solution : passer en mode **AHCI pur** dans le BIOS
- Le message kernel `ahci: Found 1 remapped NVMe devices` indique ce problème

### Flasher la clé USB d'installation
- ISO : https://www.proxmox.com/en/downloads
- Avec **Rufus** : mode GPT, UEFI, écriture en DD Image mode

### Paramètres d'installation
```
Target disk  : NVMe (nvme0n1)
Filesystem   : ext4
Hostname     : srv-proxmox-01.local
IP           : fixe (ex: 192.168.1.X/24)
```

---

## 2. Configuration post-installation

### Accès web
```
https://IP:8006
Login : root / Linux PAM
```

### Dépôts — Passer en no-subscription
```bash
# Désactiver le dépôt enterprise
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
  > /etc/apt/sources.list.d/pve-enterprise.list

# Ajouter le dépôt gratuit
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

> Le popup "No valid subscription" au login est normal et non bloquant.

### Créer un utilisateur admin (non-root)
```bash
useradd -m -s /bin/bash dani
passwd dani
usermod -aG sudo dani
pveum user add dani@pam
pveum aclmod / -user dani@pam -role Administrator
```

### Changer le hostname
```bash
hostnamectl set-hostname srv-proxmox-01

# Mettre à jour /etc/hosts
nano /etc/hosts
# Remplacer l'ancienne entrée par :
# 127.0.1.1    srv-proxmox-01.local    srv-proxmox-01

# Redémarrer les services Proxmox
systemctl restart pve-cluster pvedaemon pveproxy

# Vérifier
hostname
hostname -f
```

---

## 3. Effacement sécurisé des disques SATA

Les disques contenaient potentiellement des données professionnelles.

### Installer hdparm
```bash
apt install hdparm -y
```

### Vérifier le support Secure Erase
```bash
hdparm -I /dev/sda | grep -i erase
```

### Si Secure Erase supporté (recommandé)
```bash
hdparm -y /dev/sda
hdparm --security-set-pass TEMP /dev/sda
hdparm --security-erase TEMP /dev/sda
# Répéter pour sdb et sdc
```

### Si non supporté — remplissage par zéros
```bash
dd if=/dev/zero of=/dev/sda bs=4M status=progress
```

### Finaliser le wipe
```bash
wipefs -a /dev/sda && sgdisk --zap-all /dev/sda
wipefs -a /dev/sdb && sgdisk --zap-all /dev/sdb
wipefs -a /dev/sdc && sgdisk --zap-all /dev/sdc
```

### Vérifier que les disques sont propres
```bash
lsblk
# sda, sdb, sdc doivent apparaître sans partitions
```

---

## 4. Architecture de stockage

```
NVMe 512GB (nvme0n1)     → VMs / LXC actifs   [local-lvm, LVM-Thin]
SATA SSD x3 256GB        → Backups, ISOs, NAS  [zfs-data, RAIDZ1]
```

| Stockage | Espace utile | Redondance |
|----------|-------------|------------|
| `local-lvm` (NVMe) | ~349GB | ❌ |
| `zfs-data` (3×SATA RAIDZ1) | ~512GB | ✅ 1 disque |

---

## 5. Création du pool ZFS RAIDZ1

### Bonne pratique — utiliser les IDs disques
```bash
ls /dev/disk/by-id/ | grep -v part
```

### Créer le pool
```bash
zpool create -f zfs-data raidz1 \
  /dev/disk/by-id/ata-INTEL_SSDSC2KF256G8_SATA_256GB_BTLA75030466256CGN \
  /dev/disk/by-id/ata-INTEL_SSDSC2KF256G8_SATA_256GB_BTLA75040ANX256CGN \
  /dev/disk/by-id/ata-INTEL_SSDSC2KF256G8_SATA_256GB_BTLA750107G1256CGN
```

> ⚠️ Toujours utiliser `/dev/disk/by-id/` plutôt que `/dev/sda` pour éviter les ambiguïtés de nommage au redémarrage.

### Activer la compression
```bash
zfs set compression=lz4 zfs-data
```

### Vérifier le pool
```bash
zpool status zfs-data
zpool list
```

---

## 6. Création des datasets ZFS

```bash
zfs create zfs-data/backups   # Sauvegardes VMs/LXC
zfs create zfs-data/iso       # ISOs et templates
zfs create zfs-data/nas       # Données persistantes
```

### Vérifier
```bash
zfs list
```

---

## 7. Déclaration des stockages dans Proxmox

### Via l'UI
```
Datacenter → Storage → Add → Directory
  backups :
    ID        : backups
    Directory : /zfs-data/backups
    Content   : VZDump backup file

  iso :
    ID        : iso
    Directory : /zfs-data/iso
    Content   : ISO image, Container template
```

### Désactiver le stockage "local" par défaut
```bash
# local ne peut pas être supprimé (Proxmox le recrée automatiquement)
pvesm set local --disable 1
```

### Vérifier l'état des stockages
```bash
pvesm status
```

Résultat attendu :
```
backups    dir      active
iso        dir      active
local      dir      disabled
local-lvm  lvmthin  active
```

---

## 8. Gestion future des disques ZFS

### Remplacer un disque défaillant
```bash
zpool replace zfs-data /dev/sda /dev/sde
# ZFS rebuild automatique (resilver)
```

### Ajouter de l'espace (2ème vdev)
```bash
# Ajouter 3 nouveaux disques en 2ème RAIDZ1
zpool add zfs-data raidz1 /dev/sde /dev/sdf /dev/sdg
```

### Migration vers de plus grands disques
```bash
# 1. Créer le nouveau pool
zpool create zfs-new raidz1 /dev/sde /dev/sdf /dev/sdg

# 2. Migrer les datasets
zfs send zfs-data/backups | zfs receive zfs-new/backups
zfs send zfs-data/iso | zfs receive zfs-new/iso
zfs send zfs-data/nas | zfs receive zfs-new/nas

# 3. Détruire l'ancien pool
zpool destroy zfs-data
```

> ⚠️ ZFS RAIDZ1 ne permet pas d'étendre en remplaçant les disques un par un. Préférer des disques plus grands d'un coup.

---

## 9. Points clés et erreurs à éviter

| Problème | Cause | Solution |
|----------|-------|---------|
| NVMe non détecté | BIOS en mode Intel RST/VMD | Passer en AHCI |
| Pool ZFS introuvable à l'import | Mauvais nom ou disques non trouvés | Utiliser `/dev/disk/by-id/` |
| `local` storage ne se supprime pas | Protection Proxmox native | Le désactiver avec `pvesm set local --disable 1` |
| Erreur `in use` à la création ZFS | Ancien pool non détruit | `zpool destroy ancien-pool` puis recréer |
| Renommer un pool ZFS | `zpool export` + `zpool import ancien nouveau` | Faire avant toute utilisation |
