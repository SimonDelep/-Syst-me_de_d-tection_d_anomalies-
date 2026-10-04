# Plan de mise en place

**Projet :** Système de détection d’anomalies et de gestion de logs pour la sécurité des réseaux  
**Source :** `Projet pratique 1.pdf`  
**Socle IDS/IPS :** Wazuh  
**Cible :** une machine virtuelle Linux (Ubuntu 22.04 LTS)

Ce document est la feuille de route complète : quoi installer, dans quel ordre, quels scénarios démontrer, et quels livrables GitHub collent au barème.

---

## 1. Objectif

Mettre en place une chaîne de sécurité labo qui :

1. Collecte des logs de sécurité (syslog-ng)
2. Détecte des anomalies / tentatives d’intrusion (Wazuh)
3. Stocke et recherche ces logs (Elasticsearch)
4. Les visualise (Kibana)
5. Notifie l’administrateur dès qu’un cas confirmé est détecté (e-mail, SMS optionnel)

Le professeur exige **au moins 5 cas d’intrusion distincts**, chacun décrit, justifié, reproduit, détecté, visualisé.

## 2. Choix techniques (et pourquoi)

| Outil | Rôle dans le barème | Décision |
| --- | --- | --- |
| Wazuh Manager + Agent | IDS/IPS | Socle de détection (règles SSH, web, FIM, sudo) |
| syslog-ng | Collecteur | Obligatoire : 4 pts + 6 pts avec l’IDS |
| Elasticsearch | Stockage | Obligatoire : 6 pts avec Kibana |
| Kibana | Visualisation | Obligatoire ; dashboard **personnalisé** pour le bonus |
| Apache (ou Nginx) | Cible web | Nécessaire au scénario type PDF |
| OpenSSH | Cible brute-force | Scénario 1 |
| Script Python / intégrateur Wazuh | Alertes | E-mail requis par le PDF ; Slack = bonus |

**Point critique :** Wazuh 4.x livre aussi un *indexer* OpenSearch et un *dashboard* Wazuh. Ça ne remplace **pas** Elasticsearch + Kibana dans le barème. On installe ES + Kibana comme pile de visualisation notée. Le dashboard Wazuh est optionnel.

**Hors périmètre :** attaques hors de la VM de labo, exploits packagés contre des tiers, remplacement de Kibana par le seul dashboard Wazuh.

**Portabilité :** Elasticsearch + Kibana via Docker Compose ; Wazuh, syslog-ng, Apache et SSH en paquets sur la VM. La VM **est** le serveur labo (IP joignable). Git reconstruit ; OVA transporte la démo. Détail en sections 3 et 8.

---

## 3. Architecture

Une seule VM Linux (exigence du PDF) joue **deux rôles** : serveur SOC (Wazuh, syslog-ng, ES, Kibana) **et** cible (SSH + Apache). Ce n’est pas « une VM ou un serveur » : **la VM est le serveur**. On lui donne une IP stable ; les équipiers s’y connectent (SSH, Kibana) et lancent les 5 scénarios **contre cette IP uniquement**.

Setup par défaut (équipe pas forcément sur le même Wi‑Fi) :

1. VirtualBox + Ubuntu 22.04 chez un membre (ou un PC qui peut rester allumé).
2. **Tailscale** (gratuit) sur la VM et sur les PCs : IP type `100.x.x.x`, la même partout.
3. Export OVA pour changer de PC hôte / jour de démo.
4. GitHub pour le code.

```
  PC équipier A                    PC équipier B
  SSH / Kibana                     scripts scenarios
         \                            /
          \                          /
           v                        v
     IP labo (Tailscale 100.x ou LAN 192.168.x)
                    |
                    v
              VM Ubuntu soc-lab
     OpenSSH + Apache  |  Wazuh + syslog-ng + ES/Kibana
                    |
                    v
              alertes + dashboard + e-mail
```

**Pourquoi pas un VPS public tout de suite :** Kibana, Elasticsearch et un SSH « attaquable » sur Internet, c’est le labo qui se fait scanner par le monde entier, et l’UQAC / l’hébergeur peut couper. Tailscale = on a une IP, on se connecte, on « attaque » le labo, sans l’ouvrir au WAN.

Si un jour tout le monde est sur le même réseau (appart / labo), on peut passer la VM en **réseau ponté** et utiliser une IP `192.168.x.x` à la place de Tailscale.

**Spécifications VM recommandées**

- Ubuntu 22.04 LTS
- 8 Go RAM minimum, 4 vCPU
- 40 Go disque
- Hostname : `soc-lab`
- Carte réseau : NAT **plus** une interface Tailscale (défaut) ; ponté en option LAN
- Snapshot VirtualBox **avant** l’install, puis un second snapshot « démo »

---

## 4. Mapping barème → tâches

Le PDF affiche « Documentation GitHub : 20 points » mais liste 5 critères (5+10+5+5+5). Couvrir les **cinq** pour ne pas perdre de points.

### 4.1 Déploiement — 25 points

| Pts | Exigence | Travail concret |
| --- | --- | --- |
| 6 | IDS/IPS + syslog-ng sur l’équipement | Installer Wazuh Manager + Agent local + syslog-ng |
| 4 | syslog-ng collecte les logs de sécurité | Sources : `auth.log`, Apache, alertes Wazuh → ES |
| 6 | Elasticsearch + Kibana | Un nœud ES, Kibana `:5601`, index patterns |
| 4 | Config IDS pour détecter anomalies | `ossec.conf` + `local_rules.xml` (IDs 100xxx) |
| 5 | Intégration visualisation / réponse | Dashboard Kibana branché sur les alertes |

Configs à versionner :

- `configs/syslog-ng/syslog-ng.conf`
- `configs/wazuh/ossec.conf`
- `configs/wazuh/local_rules.xml`
- `configs/elasticsearch/elasticsearch.yml`
- `configs/kibana/kibana.yml`
- export dashboard Kibana (`.ndjson`)

### 4.2 Fonctionnalités — 45 points

| Pts | Exigence | Travail concret |
| --- | --- | --- |
| 20 | 5 scénarios complets + justification | Voir section 6 |
| 10 | Collecte des logs **prioritaires** liés aux 5 cas | Justifier auth, apache, wazuh-alerts, syscheck |
| 10 | Visualisation + explication des menaces | Dashboard + captures annotées |
| 5 | Alertes / notifications admin | E-mail SMTP Wazuh ; Slack en bonus |

### 4.3 GitHub — tous les critères listés

| Pts | Exigence | Fichier |
| --- | --- | --- |
| 5 | Structure claire | arborescence ci-dessous |
| 10 | Installation pas-à-pas | `docs/installation.md` |
| 5 | Guide d’utilisation / reproduction | `docs/utilisation.md` |
| 5 | Captures Kibana + lecture | `docs/captures/` + texte dans `docs/utilisation.md` |
| 5 | Limites, améliorations, veille | `docs/analyse.md` |

### 4.4 Bonus visés (max affiché 10, liste plus large)

| Bonus | Pts | Priorité |
| --- | --- | --- |
| Dashboard Kibana personnalisé | +5 | Haute — à faire |
| Schéma d’architecture (Mermaid) | +3 | Haute — dans le README |
| Automatisation Slack / Teams / e-mail | +5 | Haute — script `scripts/alerts/` |
| Outil complémentaire (Suricata en plus de Wazuh) | +5 | Si le temps le permet |

---

## 5. Ordre d’exécution

Faire les étapes **dans cet ordre**. Chaque étape a un critère « c’est bon quand… ».

### Phase 0 — Labo

1. Créer la VM Ubuntu 22.04, 8 Go RAM.
2. Mettre à jour le système (`apt update && apt upgrade`).
3. Hostname `soc-lab`, outils de base (`curl`, `git`, `vim`, `net-tools`).
4. Snapshot « propre ».

**OK quand :** SSH vers la VM fonctionne, `free -h` montre ≥ 8 Go.

### Phase 1 — Elasticsearch + Kibana (6 pts)

1. Installer Elasticsearch 8.x (un nœud, sécurité basique ou `xpack.security` désactivé **en labo seulement**).
2. Installer Kibana, bind `0.0.0.0` ou localhost selon comment tu présentes.
3. Créer un index de test et confirmer Discover.

**OK quand :** `http://<vm>:5601` s’ouvre et un document test est visible.

### Phase 2 — syslog-ng (4 pts)

1. Installer `syslog-ng`.
2. Sources : système + fichier `/var/log/auth.log` + logs Apache (phase 4).
3. Destination Elasticsearch (module HTTP JSON ou `elasticsearch()`).
4. Index `syslog-ng-%Y.%m.%d`.

**OK quand :** un `sudo` ou un login SSH apparaît dans Kibana via syslog-ng.

### Phase 3 — Wazuh (6 + 4 pts)

1. Installer Wazuh Manager (sans remplacer ES par l’indexer, ou indexer **en plus** seulement si la RAM le permet).
2. Installer Wazuh Agent sur la même VM, pointer vers `127.0.0.1`.
3. Dans `ossec.conf` :
   - lecture `auth.log`
   - lecture access/error Apache
   - FIM `/etc/passwd`, `/etc/shadow`, `/var/www/html`, `/tmp`
   - `email_notification` (phase 6)
4. Transférer les alertes Wazuh vers syslog-ng (fichier `alerts.json` ou sortie syslog).
5. Règles custom `local_rules.xml` pour coller aux 5 scénarios.

**OK quand :** un échec SSH génère une alerte Wazuh **et** un document dans Elasticsearch.

### Phase 4 — Cible web + règles custom

1. Installer Apache, page PHP minimale (ou un simple `index.php` qui logue la query string).
2. Activer les décodeurs Apache Wazuh.
3. Règle custom pour `exploit=`, SQLi (`' OR 1=1`), User-Agent suspect, etc.

**OK quand :** `curl "http://<IP-soc-lab>/index.php?exploit=1"` produit une alerte visible dans Kibana.

### Phase 5 — Cinq scénarios (20 + 10 pts)

Écrire un script par scénario dans `scripts/scenarios/`. Chaque script :

- génère l’événement **sur la VM**
- attend quelques secondes
- rappelle où regarder (Kibana / alerte Wazuh)

Détail des 5 cas : section 6.

**OK quand :** les 5 scripts ont chacun une alerte + un document ES + une capture.

### Phase 6 — Visualisation et notifications (10 + 5 pts)

Dashboard Kibana unique, 5 visualisations minimum :

1. Timeline des alertes
2. Top adresses IP source
3. Répartition par rule.id / description
4. Table détail (date, IP, règle, gravité)
5. Niveaux de sévérité (pie ou bar)

Notifications :

- E-mail SMTP dans Wazuh (`<email_notification>yes</email_notification>`)
- Filtrer sur priorité haute / règles custom pour éviter le spam
- Bonus : `scripts/alerts/notify.py` vers Slack ou Teams

**OK quand :** un scénario déclenche un courriel, et le dashboard n’est pas le Discover par défaut.

### Phase 7 — Dépôt GitHub

Rédiger au fil de l’eau, pas à la dernière minute. Structure :

```
README.md
PLAN.md
docs/installation.md
docs/utilisation.md
docs/scenarios.md
docs/analyse.md
docs/captures/
configs/syslog-ng/
configs/wazuh/
configs/elasticsearch/
configs/kibana/
scripts/scenarios/
scripts/alerts/
```

`README.md` : objectif, stack, schéma Mermaid, lien vers l’install, quick start.  
`docs/analyse.md` : limites (une VM, faux positifs, pas de réseau réel), améliorations (Suricata, multi-agents, SOAR), veille (XDR, OpenSearch vs Elastic).

**OK quand :** un correcteur peut cloner, suivre `docs/installation.md`, et reproduire un scénario avec `docs/utilisation.md`.

---

## 6. Les cinq scénarios

Pour chaque cas, le rapport GitHub doit contenir : **description, justification, logs collectés, règle, exemple d’alerte, lecture Kibana, commande de reproduction labo**.

### Scénario 1 — Brute-force SSH

- **Menace :** accès non autorisé par mot de passe.
- **Justification :** attaque très courante ; Wazuh a des règles natives (ex. 5710, 5551).
- **Logs :** `/var/log/auth.log` via agent + syslog-ng.
- **Repro labo :** plusieurs `ssh` avec un mauvais mot de passe vers l’IP de `soc-lab` (script, pas une attaque externe).
- **Détection :** alerte « multiple authentication failures ».

### Scénario 2 — Requête web malveillante (exemple du PDF)

- **Menace :** exploitation d’une appli web (paramètre `exploit=`, SQLi).
- **Justification :** c’est l’exemple du sujet ; montre la chaîne web → syslog-ng → Wazuh → Kibana.
- **Logs :** Apache `access.log`.
- **Repro labo :** `curl "http://<IP-soc-lab>/index.php?exploit=1"` (requête qui **matche une signature**, pas un vrai exploit).
- **Détection :** règle custom `100001` type « WEB-MISC PHP exploit attempt ».

### Scénario 3 — Atteinte à l’intégrité (FIM)

- **Menace :** modification de `/etc/passwd` ou déface du site.
- **Justification :** post-compromise classique ; force l’usage de syscheck Wazuh.
- **Logs :** alertes FIM Wazuh.
- **Repro labo :** copie de sauvegarde, petit changement contrôlé, restauration immédiate (documenter le rollback).
- **Détection :** syscheck « Integrity checksum changed ».

### Scénario 4 — Élévation de privilèges

- **Menace :** `sudo` / `su` abusif après un compte faible.
- **Justification :** chaîne kill-chain (accès → privilège).
- **Logs :** `auth.log`, éventuellement `auditd`.
- **Repro labo :** `su` avec mauvais mot de passe, ou `sudo` non autorisé.
- **Détection :** règles PAM / sudo Wazuh.

### Scénario 5 — Reconnaissance ou fichier suspect

- **Menace :** scan de ports (Nmap vers l’IP labo) **ou** dépôt d’un fichier dans `/tmp`.
- **Justification :** étape de recon / dropper ; complète les 4 cas « authentification / web / intégrité / privilège ».
- **Logs :** iptables/UFW, ou FIM `/tmp`, ou rootcheck.
- **Repro labo :** `nmap -sT <IP-soc-lab>` si les logs firewall sont branchés ; sinon création d’un fichier test sous `/tmp` surveillé.
- **Détection :** alerte scan ou « new file added ».

Ne pas mettre dans le dépôt de vrais exploits, shells inverses, ou outils d’attaque contre autre chose que la VM `soc-lab`.

---

## 7. Collecte de logs — justification (10 pts)

Ne pas tout ingérer. Justifier **uniquement** les sources liées aux 5 cas :

| Source | Scénarios | Pourquoi prioritaire |
| --- | --- | --- |
| `auth.log` | 1, 4 | Authentification SSH / sudo |
| Apache access/error | 2 | Requêtes HTTP malveillantes |
| Alertes Wazuh (`alerts.json`) | 1–5 | Événements déjà corrélés par l’IDS |
| Syscheck / FIM | 3, 5 | Intégrité fichiers |
| auditd (optionnel) | 4 | Preuve d’élévation |

Mentionner explicitement qu’on **exclut** les logs applicatifs non sécurité (cron verbeux, apt, etc.) pour réduire le bruit.

---

## 8. Serveur labo, IP, et déplacement PC → PC

Le quotidien de l’équipe : **tout le monde parle à une seule VM** (`soc-lab`) via une IP. On n’installe pas Wazuh sur chaque laptop.

| Besoin | Moyen |
| --- | --- |
| Se connecter (SSH) et voir Kibana | IP Tailscale `100.x.x.x` (défaut) ou IP LAN pontée |
| Lancer les 5 scénarios contre le serveur | Même IP, depuis le PC de l’équipier (`ssh`, `curl http://IP/`, etc.) |
| Changer de PC hôte / démo prof | Export **OVA** de cette VM |
| Recoller le projet from scratch | **GitHub** + `docker compose` + `install.sh` |

### 8.0 Accès recommandé : Tailscale

1. Créer un compte Tailscale (un pour l’équipe).
2. Installer le client sur la VM `soc-lab` → noter l’IP `100.x.x.x`.
3. Installer le client sur chaque PC Windows.
4. Vérifier : `ping 100.x.x.x`, puis `ssh user@100.x.x.x`, puis navigateur `http://100.x.x.x:5601`.

Firewall de la VM : SSH, 80/443 (Apache), 5601 (Kibana) **uniquement** sur l’interface Tailscale (ou LAN), jamais `0.0.0.0` ouvert au WAN. Elasticsearch (`9200`) reste en localhost dans Docker.

Les scénarios utilisent cette IP à la place de `127.0.0.1` (ex. `curl "http://100.x.x.x/index.php?exploit=1"`). Cible = **uniquement** `soc-lab`.

### 8.1 Ce qui va dans GitHub (reproductible)

### 8.1 Ce qui va dans GitHub (reproductible)

À la racine, tout ce qu’il faut pour refaire le stack **sans** copier le disque de la VM :

```
docker-compose.yml          # Elasticsearch + Kibana seulement
.env.example                # mots de passe / ports, jamais les secrets réels
scripts/install.sh          # paquets hôtes : syslog-ng, Wazuh, Apache, SSH
scripts/export-vm.md        # comment générer l’OVA
configs/                    # fichiers réellement utilisés
docs/installation.md        # chemin A : from scratch
docs/utilisation.md         # chemin B : importer l’OVA puis démarrer
```

Règles :

- **Aucune donnée Elasticsearch** dans Git (trop gros, inutile).
- **Aucun mot de passe** dans Git : `.env` est gitignoré, on commit `.env.example`.
- Les configs du repo = celles de la VM (copie après chaque changement).
- `install.sh` doit pouvoir tourner sur une Ubuntu 22.04 **vide** et arriver au même état.

Sur un autre PC, chemin Git :

1. Installer VirtualBox (ou VMware) + Ubuntu 22.04 (8 Go RAM).
2. `git clone` le repo dans la VM.
3. `cp .env.example .env` puis `docker compose up -d`.
4. `sudo ./scripts/install.sh`.
5. Reproduire un scénario avec `docs/utilisation.md`.

### 8.2 Ce qui va dans l’OVA (démo portable)

Quand le labo **marche**, exporter la VM entière :

- VirtualBox : Fichier → Exporter l’appliance → `.ova`
- Inclure un snapshot « démo » (services up, dashboard déjà importé)
- Copier l’OVA sur USB / Drive / NAS (souvent 8–20 Go)

Sur l’autre PC :

1. Même hyperviseur si possible (VirtualBox → VirtualBox).
2. Importer l’OVA, allouer **8 Go RAM**.
3. Démarrer, `docker compose ps` + `systemctl status wazuh-manager`.
4. Ouvrir Kibana (`http://<ip-vm>:5601`).

Documenter l’IP : en mode NAT l’IP change d’un PC à l’autre. Préférer **réseau ponté** ou noter `hostname -I` dans le guide.

### 8.3 Pièges fréquents

- Hyper-V Windows vs VirtualBox : l’OVA VirtualBox ne s’importe pas dans Hyper-V. Se mettre d’accord sur **un** outil (VirtualBox recommandé).
- CPU virtualization (VT-x/AMD-V) désactivé dans le BIOS du PC d’arrivée.
- ES/Kibana en Docker : installer Docker **dans la VM**, pas seulement sur Windows. On déplace la VM, pas les conteneurs Windows.
- Clés Wazuh agent : si on clone la VM deux fois, régénérer l’agent pour éviter les IDs dupliqués.
- Disque dynamique : l’OVA peut gonfler ; exporter depuis un snapshot propre, `apt clean`, vider `/var/log` inutiles avant export.

### 8.4 Travail d’équipe recommandé

1. Un seul membre « owner » de la VM de référence.
2. Tout le monde pousse configs/scripts sur GitHub.
3. Avant un rendu / une démo : nouvel export OVA + tag Git (`v1-demo`).
4. Les autres PCs : soit import OVA, soit rebuild via `install.sh` pour vérifier que la doc est vraie (ça vaut des points).

---

## 9. Checklist de présentation

Avant de rendre :

- [ ] Les 5 scripts produisent chacun une alerte
- [ ] Kibana montre les 5 cas sur le même dashboard
- [ ] Un e-mail part pour au moins un cas « confirmé »
- [ ] Captures d’écran datées, avec une légende (quelle règle, quelle IP)
- [ ] README + schéma Mermaid
- [ ] `docs/installation.md` relue sur une VM neuve (ou au moins un snapshot)
- [ ] Configs du repo = configs réellement utilisées sur la VM
- [ ] Snapshot « démo » pris

- [ ] OVA exportée et testée sur un **deuxième** PC
- [ ] `install.sh` + `docker compose up` validés sur une VM neuve (au moins une fois)

---

## 10. Charge de travail estimée

| Phase | Effort indicatif |
| --- | --- |
| VM + ES/Kibana | 0,5 – 1 j |
| syslog-ng → ES | 0,5 j |
| Wazuh + pont vers ES | 1 j |
| 5 scénarios + règles | 1 – 1,5 j |
| Dashboard + e-mail | 0,5 j |
| GitHub + captures + analyse | 1 j |
| **Total** | **environ 5 jours** de travail effectif |

Suricata (bonus outil complémentaire) : +0,5 j si le reste est déjà stable.
