# Troubleshoot — Perte de connectivité internet sur une VM/CT

> **Scope** : Proxmox + pfSense (router-on-a-stick) + VLANs 10–70  
> **Symptôme** : `ping 8.8.8.8` et/ou `ping google.fr` en échec depuis une VM ou CT

---

## Méthodologie

Remonter la chaîne **de la VM vers internet**, couche par couche. S'arrêter et agir dès qu'une étape échoue.

```
VM
 |
 +-- [1] A une IP ?          --> sinon : DHCP / VLAN tag Proxmox
 |
 +-- [2] Ping gateway ?       --> sinon : bridge / hookscript TAP
 |
 +-- [3] Ping 8.8.8.8 ?      --> sinon : règle firewall pfSense (protocole !)
 |
 +-- [4] Ping google.fr ?    --> sinon : DNS / PiHole / règle UDP:53
 |
 +-- [5] pfSense sort WAN ?  --> sinon : NAT Proxmox / connectivité WAN
```

---

## Étape 1 — La VM a-t-elle une IP ?

```bash
ip a
```

| Résultat | Action |
|---|---|
| Pas d'IP ou `169.254.x.x` | DHCP non reçu → vérifier pfSense > Services > DHCP Server (interface VLAN active) + VLAN tag sur l'interface VM dans Proxmox |
| IP dans `10.10.XX.100–200` | ✅ Passer à l'étape 2 |

---

## Étape 2 — La gateway est-elle joignable ?

```bash
ping 10.10.XX.1   # remplacer XX par le numéro du VLAN
```

| Résultat | Action |
|---|---|
| Timeout / 100% perte | Les paquets n'arrivent pas à pfSense → voir vérifications ci-dessous |
| Réponse OK | ✅ Passer à l'étape 3 |

### Vérifications à effectuer

**VLAN tag sur l'interface réseau de la VM (Proxmox UI) :**
- VM > Hardware > Network Device
- Bridge : `vmbr2`
- VLAN Tag : numéro du VLAN correspondant (ex: `30` pour SVC)

**vmbr2 est-il bien VLAN-aware ? (Proxmox SSH) :**

```bash
grep -A5 vmbr2 /etc/network/interfaces
# Doit contenir : bridge-vlan-aware yes
```

**TAP pfSense présents dans les bridges ? (Proxmox SSH) :**

```bash
brctl show vmbr1   # doit contenir tap100i0
brctl show vmbr2   # doit contenir tap100i1
```

> ⚠️ Si les TAP sont absents, redémarrer pfSense **proprement** depuis Proxmox (pas un reset) pour que le hookscript Perl s'exécute.

---

## Étape 3 — Internet est-il joignable ?

```bash
ping -c 4 8.8.8.8
ping -c 4 1.1.1.1
curl -s https://1.1.1.1   # contourne le blocage ICMP si besoin
```

| Résultat | Action |
|---|---|
| Timeout sur ping, curl OK | Règle firewall en **TCP only** → bloque ICMP et UDP. Passer le protocole à `any`. |
| Timeout sur tout | Règle firewall manquante ou bloc actif → voir ci-dessous |
| Réponse OK | ✅ Passer à l'étape 4 |

### Diagnostic firewall pfSense

**pfSense > Firewall > Rules > [interface VLAN]**, vérifier :

1. Existence d'une règle `pass` vers internet
2. **Protocole** : doit être `any` ou `TCP/UDP` — **jamais `TCP` seul** (bloque silencieusement ICMP et UDP)
3. **Ordre** : les règles `block` inter-VLAN doivent être placées **avant** le `pass` internet mais **après** les passes spécifiques

> 💡 **Erreur classique** : une règle `pass TCP` laisse passer HTTP/HTTPS mais bloque ping et DNS UDP. Symptôme typique : curl fonctionne, ping non.

**Logs en temps réel :**

pfSense > Status > System Logs > Firewall — filtrer par l'IP source de la VM pour voir exactement quelle règle drop les paquets.

**Packet capture (confirmation) :**

pfSense > Diagnostics > Packet Capture > interface `vtnet1.XX`

- Tu vois les ICMP echo request entrants mais pas de reply → règle firewall bloque en sortie
- Tu ne vois rien → les paquets n'arrivent même pas à pfSense → retour étape 2

---

## Étape 4 — Le DNS fonctionne-t-il ?

```bash
ping google.fr
dig @10.10.30.10 google.fr
nslookup google.fr 10.10.30.10
```

| Résultat | Action |
|---|---|
| `8.8.8.8` répond, `google.fr` non | Problème DNS pur → PiHole injoignable ou règle UDP:53 manquante |
| `dig` répond, `ping google.fr` non | DNS OK, problème de route → vérifier `ip route` dans la VM |
| Tout répond | ✅ Connectivité complète rétablie |

### Vérification règle DNS dans pfSense

Chaque VLAN doit avoir une règle autorisant l'accès au PiHole :

**pfSense > Firewall > Rules > [VLAN source]**

```
Action   : pass
Protocol : TCP/UDP
Source   : NET_[VLAN]
Dest     : PIHOLE (alias 10.10.30.10)
Port     : DNS (53)
```

> Cette règle doit être placée **avant** le bloc général inter-VLAN.

---

## Étape 5 — pfSense sort-il sur le WAN ?

À effectuer si tout semble correct côté VLAN mais que la VM ne sort toujours pas.

**pfSense > Diagnostics > Ping :**
- Interface source : `WAN (vtnet0)`
- Destination : `8.8.8.8`

| Résultat | Action |
|---|---|
| pfSense ne ping pas `8.8.8.8` | Problème en amont → vérifier connectivité WAN Proxmox (ci-dessous) |
| pfSense ping OK mais VM non | Problème isolé dans les règles firewall du VLAN → reprendre étape 3 |

### Vérifications Proxmox (WAN WiFi temporaire)

```bash
ip route show table 100
# Doit contenir : default via 172.20.10.1 dev wlp4s0

ip rule list
# Doit contenir : from 10.10.0.2 lookup 100

iptables -t nat -L POSTROUTING -n
# Doit contenir MASQUERADE sur wlp4s0 pour 10.10.0.0/30 et 10.10.0.0/8
```

---

## Tableau des erreurs classiques

| Symptôme | Cause | Correctif |
|---|---|---|
| `ping` et DNS KO, `curl` OK | Règle firewall `TCP` only | Passer le protocole à `any` dans pfSense |
| Pas d'IP DHCP | VLAN tag absent ou incorrect sur la VM | Proxmox > VM > Hardware > Network : vérifier `vmbr2` + VLAN Tag |
| Gateway KO après reboot pfSense | TAP non bridgées (hookscript non exécuté) | Redémarrer pfSense proprement depuis Proxmox |
| DNS KO malgré internet OK | Règle UDP:53 vers PiHole manquante pour ce VLAN | Ajouter règle `pass TCP/UDP > PIHOLE:53` dans pfSense |
| Tout bloqué malgré règles pass | Règle `block` placée avant la règle `pass` | Réordonner : `pass` en premier, `block all` en dernier |
| pfSense ne sort pas sur WAN | Route default Proxmox cassée ou NAT absent | Vérifier `ip route table 100` et `iptables POSTROUTING` |

---

## Commandes de référence rapide

### Depuis la VM/CT (Linux)

```bash
ip a                              # vérifier IP + interface
ip route                          # vérifier route par défaut
ping -c 4 10.10.XX.1             # tester gateway
ping -c 4 8.8.8.8               # tester internet (ICMP)
curl -s https://1.1.1.1          # tester internet (TCP/443)
dig @10.10.30.10 google.fr       # tester DNS PiHole
nslookup google.fr 10.10.30.10   # alternative DNS
```

### Depuis Proxmox (SSH)

```bash
brctl show vmbr2                  # vérifier TAP dans bridge
ip route show table 100           # vérifier route WAN
ip rule list                      # vérifier policy routing
iptables -t nat -L POSTROUTING -n # vérifier NAT MASQUERADE
cat /etc/network/interfaces       # vérifier config bridges
```

### Depuis pfSense (GUI)

| Où | Quoi |
|---|---|
| Diagnostics > Ping | Tester depuis `vtnet0` (WAN) vers `8.8.8.8` |
| Firewall > Rules > [VLAN] | Vérifier règles et **protocoles** |
| Status > System Logs > Firewall | Voir paquets bloqués en temps réel |
| Diagnostics > Packet Capture | Capturer trafic entrant sur `vtnet1.XX` |
| Services > DHCP Server > [VLAN] | Vérifier étendue DHCP active |
