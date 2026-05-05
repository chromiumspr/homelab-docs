# Configuration Proxmox initiale

## 1. Créer un utilisateur admin (non-root)
```bash
useradd -m -s /bin/bash dani
passwd dani
usermod -aG sudo dani
pveum user add dani@pam
pveum aclmod / -user dani@pam -role Administrator
```

## 2. Changer le hostname
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
