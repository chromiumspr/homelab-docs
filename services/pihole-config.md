# Déploiement PiHole — VLAN SVC

**VM :** `vm-svc-01` | **IP :** `10.10.30.10` | **Interface web :** `http://10.10.30.10:8080/admin`

---

## Rôle de PiHole dans l'architecture

PiHole est le **DNS resolver central** de tout le homelab. Toutes les requêtes DNS de tous les VLANs transitent par lui via pfSense, ce qui permet :

- **Filtrage publicitaire et malware** à l'échelle du réseau entier
- **Visibilité** sur toutes les résolutions DNS (query log)
- **DNS interne** — résolution de noms locaux (`proxmox.home.lab`, etc.)

---

## Flux DNS complet

```
Client (n'importe quel VLAN)
        │
        │  requête DNS (port 53)
        ▼
pfSense — gateway du VLAN (ex: 10.10.30.1)
        │
        │  forwarding (DNS Resolver en mode forwarding)
        ▼
PiHole (10.10.30.10:53)
        │
        ├─ domaine dans la blocklist ? → réponse 0.0.0.0 (bloqué)
        │
        └─ domaine autorisé ?
                │
                │  résolution récursive
                ▼
           Serveurs DNS upstream (1.1.1.1, 8.8.8.8)
                │
                ▼
           Réponse remontée jusqu'au client
```

**Exemple concret** — ton PC demande `google.com` :

1. PC → `10.10.10.1` (pfSense MGMT) sur port 53
2. pfSense → `10.10.30.10` (PiHole) sur port 53
3. PiHole vérifie ses blocklists → `google.com` n'est pas bloqué
4. PiHole → `1.1.1.1` → obtient `142.251.142.14`
5. Réponse remonte jusqu'au PC

**Exemple bloqué** — ton PC charge une page avec une pub `ads.doubleclick.net` :

1. PC → pfSense → PiHole
2. PiHole vérifie → `ads.doubleclick.net` est dans la blocklist
3. PiHole répond `0.0.0.0` → la pub ne charge pas
4. Le compteur "Queries blocked" dans le dashboard s'incrémente

---

## Configuration pfSense

Dans pfSense → **Services → DNS Resolver → General Settings** :

- **Enable** : ✅
- **Enable Forwarding Mode** : ✅ — pfSense forwarde vers PiHole au lieu de résoudre lui-même

Dans pfSense → **System → General Setup** → DNS Servers :

```
10.10.30.10    (PiHole)
```

Cela force pfSense à utiliser PiHole comme seul upstream DNS pour tous les VLANs.

---

## Docker Compose

Fichier : `/opt/docker/pihole/docker-compose.yml`

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    environment:
      TZ: Europe/Paris
      WEBPASSWORD: "ton_mot_de_passe"
    volumes:
      - ./data/etc-pihole:/etc/pihole
      - ./data/etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    cap_add:
      - NET_ADMIN
```

### Explication des paramètres

**`image: pihole/pihole:latest`**  
Image officielle PiHole depuis Docker Hub. `latest` = dernière version stable.

**`restart: unless-stopped`**  
Le container redémarre automatiquement après un reboot de la VM, sauf s'il a été arrêté manuellement avec `docker compose down`.

**`environment`**  
- `TZ` : fuseau horaire pour les timestamps dans les logs et le query log
- `WEBPASSWORD` : mot de passe de l'interface web admin

**`volumes`**  
Montages persistants — les données survivent aux redémarrages et mises à jour du container :
- `./data/etc-pihole` → configuration PiHole (blocklists, DNS custom, etc.)
- `./data/etc-dnsmasq.d` → configuration dnsmasq avancée (DNS local, DHCP si utilisé)

**`ports`**  
- `53:53/tcp` et `53:53/udp` → exposition du DNS sur l'IP de la VM (`10.10.30.10:53`)
- `8080:80/tcp` → interface web accessible sur `http://10.10.30.10:8080/admin`

**`cap_add: NET_ADMIN`**  
PiHole a besoin de capabilities réseau avancées pour gérer dnsmasq (manipulation d'interfaces, gestion DHCP si activé).

---

## Déploiement

```bash
cd /opt/docker/pihole
docker compose up -d
docker ps
```

Vérifier que PiHole répond :

```bash
dig google.com @10.10.30.10
```

Réponse attendue : `status: NOERROR` avec une IP dans la section ANSWER.

---

## Réinitialiser le mot de passe admin

```bash
docker exec -it pihole pihole setpassword
```

---

## Configuration PiHole post-déploiement

Dans l'interface web → **Settings → DNS** :

- **Upstream DNS** : cocher Cloudflare (`1.1.1.1`) et/ou Google (`8.8.8.8`)
- **Interface settings** : `Permit all origins` — nécessaire pour accepter les requêtes venant de pfSense (`10.10.30.1`) qui n'est pas sur le même subnet que le container

---

## Structure des données

```
/opt/docker/pihole/
├── docker-compose.yml
└── data/
    ├── etc-pihole/          # config principale, blocklists, whitelist
    └── etc-dnsmasq.d/       # config dnsmasq avancée
```

Ces dossiers sont créés automatiquement au premier `docker compose up -d`.  
Ce sont les seuls répertoires à inclure dans les sauvegardes Restic.

---

## Commandes utiles

```bash
# Voir les logs en temps réel
docker logs -f pihole

# Mettre à jour PiHole
docker compose pull && docker compose up -d

# Vérifier les stats depuis le CLI
docker exec -it pihole pihole status
docker exec -it pihole pihole -c    # chronometer
```

---

## Notes

- PiHole écoute sur `0.0.0.0:53` — il répond à toutes les interfaces de la VM
- Le query log est accessible dans l'interface web → **Query Log** en temps réel
- Les blocklists se mettent à jour automatiquement (configurable dans **Settings → Blocklists**)
- Ne pas exposer le port 53 sur Internet — la règle pfSense SVC bloque déjà l'accès externe
