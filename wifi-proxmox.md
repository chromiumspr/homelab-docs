# Connexion WiFi sur Proxmox (sans câble Ethernet)

> Documentation issue d'une session de dépannage sur `srv-proxmox-01`  
> Carte WiFi : `wlp4s0` (Intel iwlwifi)

---

## Prérequis

Les outils WiFi ne sont pas inclus par défaut dans Proxmox. Il faut les installer manuellement via une clé USB ou un partage USB depuis un téléphone.

### Paquets nécessaires

À télécharger depuis une machine Debian/Ubuntu connectée :

```bash
apt download libnl-3-200 libnl-genl-3-200 libnl-route-3-200 adduser iw wpasupplicant
```

### Installation sur Proxmox

Copier les `.deb` sur Proxmox puis installer dans cet ordre :

```bash
export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin

dpkg -i libnl-3-200_*.deb
dpkg -i libnl-genl-3-200_*.deb
dpkg -i libnl-route-3-200_*.deb
dpkg -i adduser_*.deb
dpkg -i iw_*.deb
dpkg -i wpasupplicant_*.deb
```

> **Note :** Le `export PATH` est nécessaire car Proxmox n'inclut pas `/usr/sbin` et `/sbin` par défaut dans le PATH de root.

---

## Configuration initiale

### Créer le fichier wpa_supplicant

```bash
wpa_passphrase "NomDuRéseau" "MotDePasse" > /etc/wpa_supplicant.conf
echo "p2p_disabled=1" >> /etc/wpa_supplicant.conf
```

> `p2p_disabled=1` est indispensable pour éviter des erreurs liées au WiFi Direct (P2P) qui empêchent la connexion.

Le fichier `/etc/wpa_supplicant.conf` doit ressembler à :

```
network={
    ssid="NomDuRéseau"
    #psk="MotDePasseEnClair"   # commenté, ignoré
    psk=aabbcc1122...          # hash PSK utilisé
}
p2p_disabled=1
```

---

## Connexion WiFi

### Désactiver le service systemd (évite les conflits)

```bash
systemctl stop wpa_supplicant
systemctl disable wpa_supplicant
```

### Démarrer la connexion

```bash
ip link set wlp4s0 up
wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf
dhclient wlp4s0
```

### Vérifier la connexion

```bash
ip a show wlp4s0
ping -c 3 8.8.8.8
```

### Arrêter proprement la connexion

```bash
dhclient -r wlp4s0
pkill wpa_supplicant
ip link set wlp4s0 down
```

---

## Aliases (raccourcis)

Ajouter dans `/root/.bashrc` et `/root/.bash_profile` :

```bash
# Dans .bashrc
alias wifi-start='ip link set wlp4s0 up && wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf && dhclient wlp4s0'
alias wifi-stop='dhclient -r wlp4s0 && pkill wpa_supplicant && ip link set wlp4s0 down'

# Dans .bash_profile (pour que .bashrc soit chargé au login)
source ~/.bashrc
```

Appliquer sans reboot :

```bash
source ~/.bashrc
```

Utilisation :

```bash
wifi-start   # démarre la connexion
wifi-stop    # arrête proprement
```

---

## Gestion des réseaux

### Ajouter un réseau supplémentaire

```bash
wpa_passphrase "NouveauRéseau" "NouveauMotDePasse" >> /etc/wpa_supplicant.conf
```

> Utiliser `>>` pour **ajouter** sans écraser. Vérifier que `p2p_disabled=1` est toujours présent à la fin du fichier.

### Remplacer complètement le réseau

```bash
wpa_passphrase "NouveauRéseau" "NouveauMotDePasse" > /etc/wpa_supplicant.conf
echo "p2p_disabled=1" >> /etc/wpa_supplicant.conf
```

### Supprimer un réseau manuellement

```bash
nano /etc/wpa_supplicant.conf
# Supprimer le bloc network={...} correspondant
# Ctrl+O pour sauvegarder, Ctrl+X pour quitter
```

### Définir une priorité entre plusieurs réseaux

```
network={
    ssid="Réseau-Principal"
    psk=aabbcc...
    priority=10
}
network={
    ssid="Réseau-Secondaire"
    psk=ddeeff...
    priority=5
}
p2p_disabled=1
```

> Plus le chiffre est élevé, plus le réseau est prioritaire.

### Appliquer les changements

```bash
wifi-stop && wifi-start
```

---

## Dépannage

### Interface en état DOWN

```bash
ip link set wlp4s0 up
```

### Vérifier que le firmware est chargé

```bash
dmesg | grep iwlwifi
```

### Vérifier l'état de la connexion

```bash
wpa_cli -i wlp4s0 status
```

Résultat attendu : `wpa_state=COMPLETED`

### Plusieurs instances wpa_supplicant en conflit

```bash
pkill -9 wpa_supplicant
systemctl stop wpa_supplicant
ps aux | grep wpa_supplicant  # vérifier
```

### Pas de résolution DNS

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

---

## Notes importantes

- Le WiFi sur Proxmox est une solution **temporaire**. Les hyperviseurs nécessitent une connexion filaire stable pour les VMs et les bridges réseau.
- Le `vmbr0` est configuré sur `192.168.10.10/24` — s'assurer que le réseau WiFi utilise un subnet différent pour éviter les conflits de routage.
- La connexion WiFi **ne survit pas à un reboot** sans configuration supplémentaire dans `/etc/network/interfaces`.
