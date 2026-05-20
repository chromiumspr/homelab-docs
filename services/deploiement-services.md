# Logique de déploiement d'un nouveau service

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Étape 1 — Choisir le VLAN d'hébergement](#2-étape-1--choisir-le-vlan-dhébergement)
3. [Étape 2 — Déployer le conteneur](#3-étape-2--déployer-le-conteneur)
4. [Étape 3 — Configurer l'accès réseau](#4-étape-3--configurer-laccès-réseau)
5. [Étape 4 — Vérification](#5-étape-4--vérification)
6. [Référence — règles firewall types](#6-référence--règles-firewall-types)

---

## 1. Vue d'ensemble

Tout nouveau service suit le même processus en 4 étapes :

```
Choisir le VLAN → Déployer le conteneur → Configurer l'accès → Vérifier
```

Le point d'entrée pour tous les services accessibles depuis le LAN est **NPM** (`vm-svc-01`, `10.10.30.10`). Les services d'administration restent accessibles directement depuis MGMT uniquement.

---

## 2. Étape 1 — Choisir le VLAN d'hébergement

| VLAN | Subnet | Usage |
|---|---|---|
| `MGMT (10)` | `10.10.10.0/24` | Outils d'administration (Portainer, Prometheus) |
| `SVC (30)` | `10.10.30.0/24` | Services accessibles depuis le LAN (NPM, PiHole, Grafana) |
| `FUNLAB (60)` | `10.10.60.0/24` | Stack média (*arr, qBittorrent, Homarr, Bookstack) |
| `BCKP (70)` | `10.10.70.0/24` | Backup uniquement (PBS) |

---

## 3. Étape 2 — Déployer le conteneur

Sur la VM du VLAN cible, créer le compose dans `/opt/docker/<service>/` :

```bash
mkdir -p /opt/docker/<service>
nano /opt/docker/<service>/docker-compose.yml
cd /opt/docker/<service>
docker compose up -d
```

---

## 4. Étape 3 — Configurer l'accès réseau

Selon le besoin d'accès, trois cas possibles :

### Cas A — Accès depuis le LAN via NPM

Le flux est :
```
Client (LAN) → NPM:80/443 (SVC) → Service:PORT (VLAN cible)
```

| Étape | Où | Action |
|---|---|---|
| **DNS** | PiHole | Ajouter `<service>.home.lab` → `10.10.30.10` (IP NPM) |
| **Proxy** | NPM | Créer Proxy Host : `<service>.home.lab` → `<IP_service>:<PORT>` |
| **Firewall** | pfSense VLAN30SVC | Ajouter règle `NET_SVC → <IP_service>:<PORT>` **au-dessus** des règles deny |

> ⚠️ L'ordre des règles pfSense est critique. La règle `pass` doit toujours être **avant** le `blck to vlan X` correspondant.

### Cas B — Accès depuis MGMT uniquement

Le flux est :
```
Admin (MGMT) → Service:PORT (VLAN cible)
```

| Étape | Où | Action |
|---|---|---|
| **DNS** | PiHole | Ajouter `<service>.home.lab` → `<IP_service>` directement |
| **Firewall** | pfSense VLAN10MGMT | Ajouter règle `NET_MGMT → <IP_service>:<PORT>` |

Pas de NPM — accès direct, réservé aux outils d'administration.

### Cas C — Double accès (pattern admin + utilisateur)

Certains services exposent deux niveaux d'accès selon l'origine :

| Niveau | Accès | Depuis |
|---|---|---|
| Utilisateur / read-only | `<service>.home.lab` via NPM | LAN |
| Admin | `<IP_service>:<PORT>` direct ou WireGuard admin | MGMT uniquement |

Exemples appliqués :

| Service | LAN (via NPM) | MGMT (direct) |
|---|---|---|
| Grafana | Dashboard anonyme read-only | Admin `10.10.10.20:3000` |
| Portainer | Vue lecture via proxy | Admin `10.10.10.20:9000` |

---

## 5. Étape 4 — Vérification

```bash
# Résolution DNS
nslookup <service>.home.lab

# Connectivité directe depuis la VM source
curl -I http://<IP_service>:<PORT>

# Test via proxy NPM
curl -I http://<service>.home.lab
```

En cas d'échec, vérifier dans pfSense **Firewall → Logs** pour identifier la règle qui bloque.

Consulter aussi `troubleshoot-connectivite-vm.md` pour la méthodologie complète.

---

## 6. Référence — règles firewall types

| Flux | Interface pfSense | Source | Destination | Port |
|---|---|---|---|---|
| LAN → NPM | `VLAN20LAN` | `NET_LAN` | `NPM_INT` | `80, 443` |
| MGMT → NPM | `VLAN10MGMT` | `NET_MGMT` | `NPM_INT` | `80, 443` |
| SVC → service MGMT | `VLAN30SVC` | `NET_SVC` | `<IP_service>` | `<PORT>` |
| SVC → service FUNLAB | `VLAN30SVC` | `NET_SVC` | `<IP_service>` | `<PORT>` |
| MGMT → agent Portainer | `VLAN10MGMT` | `NET_MGMT` | `<IP_VM>` | `9001` |
| MGMT → Prometheus | `VLAN10MGMT` | `NET_MGMT` | `10.10.10.20` | `9090` |
| SVC → Prometheus (scrape) | `VLAN30SVC` | `NET_SVC` | `10.10.10.20` | `9090` |

> 💡 Toujours créer un **alias pfSense** pour les IPs et ports récurrents (ex. `PORTAINER`, `NPM_INT`, `PIHOLE`) pour faciliter la maintenance des règles.
