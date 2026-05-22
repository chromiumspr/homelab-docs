# Installation Docker 

---

## 1. Installation de Docker

Docker fournit un script d'installation officiel qui détecte automatiquement la distribution et configure les dépôts.

```bash
curl -fsSL https://get.docker.com | sh
```

Ce script effectue les opérations suivantes :
- Ajoute le dépôt officiel Docker (`download.docker.com`)
- Installe `docker-ce`, `docker-ce-cli`, `containerd.io` et `docker-compose-plugin`

---

## 2. Ajout de l'utilisateur au groupe Docker

Par défaut, seul `root` peut interagir avec le daemon Docker. Pour permettre à `dani` d'utiliser Docker sans `sudo` :

```bash
usermod -aG docker dani
```

> La modification ne prend effet qu'après reconnexion SSH. Déconnecte-toi et reconnecte-toi pour que le groupe soit actif.

---

## 3. Activation et démarrage du service

```bash
systemctl enable --now docker
```

- `enable` : Docker démarrera automatiquement au boot
- `--now` : démarre le service immédiatement

---

## 4. Vérification

```bash
docker --version
systemctl status docker
docker run hello-world
```

La commande `hello-world` télécharge une image de test et affiche un message de confirmation. C'est le moyen le plus simple de valider que Docker fonctionne end-to-end.

---

## 5. Structure des répertoires

Tous les services Docker sont organisés sous `/opt/docker/`, avec un sous-dossier par service :

```
/opt/docker/
├── pihole/
│   ├── docker-compose.yml
│   ├── .env               # variables d'environnement (optionnel)
│   └── data/              # volumes persistants
├── npm/
│   ├── docker-compose.yml
│   └── data/
└── grafana/
    ├── docker-compose.yml
    └── data/
```

**Pourquoi cette structure ?**
- Chaque service est indépendant — `docker compose up -d` dans son dossier suffit
- Les volumes sont locaux dans `data/` — sauvegarde Restic ciblée par service
- Lisibilité et maintenance simplifiées à mesure que les services s'accumulent

Créer la structure initiale :

```bash
mkdir -p /opt/docker/pihole
mkdir -p /opt/docker/npm
mkdir -p /opt/docker/grafana
```

---

## 6. Commandes Docker Compose essentielles

| Commande | Description |
|---|---|
| `docker compose up -d` | Démarre les services en arrière-plan |
| `docker compose down` | Arrête et supprime les containers |
| `docker compose restart` | Redémarre les services |
| `docker compose logs -f` | Affiche les logs en temps réel |
| `docker ps` | Liste les containers en cours d'exécution |
| `docker exec -it <nom> bash` | Ouvre un shell dans un container |

---

## Notes

- Docker stocke ses données dans `/var/lib/docker/` — surveiller l'espace disque
- Les images non utilisées s'accumulent, nettoyer périodiquement avec `docker system prune`
- Le plugin `docker-compose-plugin` est inclus dans l'installation — utiliser `docker compose` (avec espace) et non l'ancien `docker-compose` (avec tiret)
