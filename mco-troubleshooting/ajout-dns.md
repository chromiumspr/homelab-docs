# Procédure — Ajouter un nouveau service (DNS + Proxy)

Cette procédure s'applique à chaque fois qu'un nouveau service est déployé et doit être accessible depuis le LAN via une URL propre.

---

## Vue d'ensemble

Chaque service accessible depuis le LAN nécessite 3 étapes :

```
1. Enregistrement DNS dans PiHole    →  résolution du nom
2. Proxy host dans NPM               →  routage vers le bon backend
3. Règle firewall pfSense (si besoin) →  autorisation du flux
```

---

## Étape 1 — Enregistrement DNS dans PiHole

Dans l'interface PiHole → **Local DNS → DNS Records → Add** :

| Champ | Valeur |
|---|---|
| Domain | `monservice.home.lab` |
| IP Address | `10.10.30.10` (IP de NPM) |

> **Toujours pointer vers l'IP de NPM** (`10.10.30.10`), jamais directement vers l'IP du service backend. C'est NPM qui fait le routage selon le nom de domaine.

**Exceptions** — pointer directement vers l'IP du service si :
- Le service n'est pas proxifié (accès admin direct, ex. switch, Proxmox depuis MGMT)
- Le service écoute sur un port non standard non géré par NPM

### Exemple

```
pihole.home.lab    →  10.10.30.10  (NPM, pas PiHole directement)
npm.home.lab       →  10.10.30.10
grafana.home.lab   →  10.10.30.10
homarr.home.lab    →  10.10.30.10
proxmox.home.lab   →  10.10.10.10  (accès direct, pas via NPM)
```

---

## Étape 2 — Proxy host dans NPM

Dans NPM → **Proxy Hosts → Add Proxy Host** :

**Onglet Details**

| Champ | Valeur |
|---|---|
| Domain Names | `monservice.home.lab` |
| Scheme | `http` (ou `https` si le backend exige TLS) |
| Forward Hostname | IP réelle du service backend |
| Forward Port | Port réel du service backend |
| Cache Assets | ✅ |
| Block Common Exploits | ✅ |
| Websockets Support | ✅ si interface interactive |

**Onglet Advanced** (si redirection de path nécessaire)

```nginx
location = / {
    return 301 /chemin/voulu;
}
```

### Tableau de référence des backends

| Service | Forward Hostname | Forward Port | Scheme |
|---|---|---|---|
| PiHole | `10.10.30.10` | `8080` | http |
| NPM | `10.10.30.10` | `81` | http |
| pfSense | `10.10.10.1` | `443` | https |
| Grafana | `10.10.30.10` | `3000` | http |
| Homarr | `10.10.60.x` | `7575` | http |
| Bookstack | `10.10.60.x` | `6875` | http |

---

## Étape 3 — Règle firewall pfSense

### Cas 1 — Service dans SVC (même VLAN que NPM)
Aucune règle inter-VLAN nécessaire — NPM et le service sont dans le même VLAN, le trafic ne passe pas par pfSense.

### Cas 2 — Service dans FUNLAB ou autre VLAN
NPM doit pouvoir joindre le backend dans un autre VLAN. Ajouter dans pfSense → **Firewall → Rules → VLAN30SVC** :

```
Action   : Pass
Protocol : TCP
Source   : NPM_SVC
Dest     : IP du service
Port dst : Port du service
Descr    : NPM to <nom service>
```

### Cas 3 — Nouveau VLAN source (autre que LAN)
Si des clients d'un autre VLAN doivent accéder au service via NPM, ajouter dans le VLAN source :

```
Action   : Pass
Protocol : TCP
Source   : NET_<VLAN>
Dest     : NPM_SVC
Port dst : WEB (80, 443)
Descr    : <VLAN> to NPM proxy
```

---

## Checklist complète

```
□ Enregistrement DNS ajouté dans PiHole (→ 10.10.30.10)
□ Proxy host créé dans NPM (domaine + backend + port)
□ Règle pfSense ajoutée si service dans autre VLAN
□ Test depuis le PC : http://monservice.home.lab
□ Vérification dans PiHole Query Log (requête DNS visible)
□ Vérification dans NPM Logs (requête HTTP visible)
```

---

## Vérification et debug

**Le nom ne résout pas**
```bash
# Depuis le PC
nslookup monservice.home.lab
# Doit retourner 10.10.30.10
```
→ Vérifier l'enregistrement DNS dans PiHole

**Le nom résout mais page inaccessible**
```bash
# Depuis le PC
curl -v http://monservice.home.lab
```
→ Vérifier le proxy host NPM (bon port ? bon hostname ?)
→ Vérifier la règle pfSense si service dans autre VLAN

**Erreur 502 Bad Gateway**
→ NPM ne joint pas le backend — vérifier que le service tourne (`docker ps`)
→ Vérifier la règle pfSense SVC → VLAN du service

**Erreur 403 Forbidden**
→ Le backend répond mais refuse la requête — souvent un problème de path
→ Utiliser l'onglet Advanced de NPM pour ajouter une redirection

---

## Notes

- Toujours tester la résolution DNS avant de debugger NPM
- NPM régénère ses configs à chaque modification UI — les édits manuels dans `/data/nginx/proxy_host/` sont perdus
- Pour les services FUNLAB, penser à ajouter la règle pfSense `SVC → FUNLAB` au port concerné
