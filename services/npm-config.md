# Déploiement NPM (Nginx Proxy Manager) — VLAN SVC

**VM :** `vm-svc-01` | **IP :** `10.10.30.10` | **Interface admin :** `http://npm.home.lab` (port 81)

---

## Rôle de NPM dans l'architecture

NPM est le **reverse proxy interne** du homelab. Il est le point d'entrée unique pour tous les services accessibles depuis le LAN, ce qui permet :

- **URL propres** — `pihole.home.lab` au lieu de `10.10.30.10:8080`
- **Centralisation** — une seule IP et un seul port 80/443 exposé au LAN
- **Sécurité** — les ports applicatifs des services ne sont jamais exposés directement
- **SSL** — terminaison TLS centralisée (certificats Let's Encrypt ou auto-signés)

---

## Flux d'une requête via NPM

```
Client LAN (PC, téléphone)
        │
        │  http://pihole.home.lab
        ▼
pfSense — règle LAN → NPM_SVC:80/443
        │
        ▼
NPM (10.10.30.10:80)
        │
        │  NPM lit le header "Host: pihole.home.lab"
        │  et route vers le bon backend
        ▼
PiHole (10.10.30.10:8080)
        │
        ▼
Réponse remontée jusqu'au client
```

---

## Docker Compose

Fichier : `/opt/docker/npm/docker-compose.yml`

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./data/letsencrypt:/etc/letsencrypt
```

### Explication des paramètres

**`ports`**
- `80:80` — trafic HTTP entrant (redirections, requêtes non-SSL)
- `443:443` — trafic HTTPS entrant (services avec certificat SSL)
- `81:81` — interface admin NPM (à ne pas exposer sur Internet)

**`volumes`**
- `./data` — configuration des proxy hosts, certificats, logs
- `./data/letsencrypt` — certificats Let's Encrypt

---

## Déploiement

```bash
cd /opt/docker/npm
docker compose up -d
docker ps
```

### Credentials par défaut

```
Email    : admin@example.com
Password : changeme
```

NPM force le changement à la première connexion.

---

## Règle pfSense requise

Dans pfSense → **Firewall → Rules → VLAN20LAN** :

```
Action   : Pass
Protocol : TCP
Source   : NET_LAN
Dest     : NPM_SVC
Port dst : WEB (80, 443)
Descr    : LAN to NPM proxy
```

---

## Ajouter un proxy host

Dans NPM → **Proxy Hosts → Add Proxy Host** :

| Champ | Valeur |
|---|---|
| Domain Names | `service.home.lab` |
| Scheme | `http` ou `https` selon le backend |
| Forward Hostname | IP du service backend |
| Forward Port | Port du service backend |
| Cache Assets | ✅ recommandé |
| Block Common Exploits | ✅ toujours |
| Websockets Support | ✅ si console interactive (Proxmox, etc.) |

---

## Services actuellement proxifiés

| Domaine | Backend | Port |
|---|---|---|
| `pihole.home.lab` | `10.10.30.10` | `8080` |
| `npm.home.lab` | `10.10.30.10` | `81` |
| `pfsense.home.lab` | `10.10.10.1` | `443` |

---

## Redirection personnalisée (exemple PiHole)

NPM ne supporte pas nativement les redirections de path dans son interface. Pour forcer `pihole.home.lab` → `/admin/login`, éditer le fichier de config généré :

```bash
docker exec -it npm bash
apt-get update && apt-get install -y nano
nano /data/nginx/proxy_host/1.conf
```

Remplacer le bloc `location / {` par :

```nginx
location = / {
    return 301 /admin/login;
}

location / {
    # Proxy!
    include conf.d/include/proxy.conf;
}
```

Puis recharger Nginx :

```bash
nginx -s reload
```

> **Attention** — cette modification est écrasée si le proxy host est modifié depuis l'interface NPM. À refaire après chaque modification.

---

## Structure des données

```
/opt/docker/npm/
├── docker-compose.yml
└── data/
    ├── nginx/
    │   └── proxy_host/    # configs générées par NPM (1.conf, 2.conf...)
    ├── logs/              # access et error logs par proxy host
    └── letsencrypt/       # certificats SSL
```

---

## Commandes utiles

```bash
# Voir les logs NPM
docker logs -f npm

# Mettre à jour NPM
docker compose pull && docker compose up -d

# Recharger Nginx sans redémarrer le container
docker exec -it npm nginx -s reload

# Accéder au shell NPM
docker exec -it npm bash
```

---

## Notes

- Le port 81 (interface admin) ne doit jamais être exposé sur Internet
- NPM régénère les fichiers `.conf` dans `/data/nginx/proxy_host/` à chaque modification depuis l'UI — les éditions manuelles sont perdues
- Pour des redirections permanentes, privilégier l'onglet **Advanced** du proxy host quand disponible
- NPM supporte Let's Encrypt nativement pour les domaines publics — pour `home.lab` (domaine local) utiliser des certificats auto-signés ou un certificat wildcard DNS-01
