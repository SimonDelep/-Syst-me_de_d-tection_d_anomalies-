# IDS-Lab — Projet pratique 1 (Maîtrise en cybersécurité)

Système de détection d'intrusion basé sur **Suricata** (IDS/IPS), **syslog-ng** et la pile **ELK** (Elasticsearch/Kibana), déployé sur une VM Ubuntu Server.

> **Note importante** : Snort (2 ou 3) n'est plus disponible dans les dépôts d'Ubuntu 26.04 ("resolute"). Le projet utilise donc **Suricata**, le successeur de facto de Snort, activement maintenu, compatible avec le même format de règles (Emerging Threats Open / Snort).

---

## 1. Vue d'ensemble de l'architecture

```
Trafic réseau (enp0s3)
        │
        ▼
   ┌─────────┐        eve.json        ┌───────────┐        ┌───────────────┐        ┌────────┐
   │ Suricata │ ─────────────────────▶ │ Filebeat  │ ─────▶ │ Elasticsearch │ ─────▶ │ Kibana │
   │  (IDS)   │                        │ (module   │        │               │        │        │
   └─────────┘                        │ suricata) │        └───────────────┘        └────────┘
        │                              └───────────┘               ▲
        │                                                          │
   Logs système                                                    │
   (auth, kernel...)                                               │
        │                                                          │
        ▼                                                          │
   ┌──────────┐          index syslog-ng          ┌────────────────┘
   │ syslog-ng │ ───────────────────────────────▶ │
   └──────────┘
```

| Composant | Rôle | Port |
|---|---|---|
| Suricata 8.0.3 | Détection d'intrusion (IDS), 52 888 règles Emerging Threats Open | — (écoute passive sur `enp0s3`) |
| syslog-ng 4.8.1 | Collecte des logs système → Elasticsearch | — |
| Filebeat 8.19.21 | Parsing du format EVE JSON de Suricata → Elasticsearch | — |
| Elasticsearch 8.19.21 | Stockage et indexation | 9200 (HTTPS) |
| Kibana 8.19.21 | Visualisation, dashboards, Discover | 5601 (HTTP) |

---

## 2. Prérequis machine

- VM Ubuntu Server 26.04.1 ("resolute")
- 4 à 5 Go de RAM minimum (Elasticsearch + Kibana + Suricata + Filebeat + syslog-ng ensemble sont gourmands — un OOM kill a déjà été observé avec moins de RAM disponible)
- 40 Go de disque
- Réseau en mode **NAT**, interface `enp0s3`

### Redirections de port à configurer (VirtualBox, côté hôte)

Ces réglages ne sont **pas inclus** dans un export OVA — à recréer manuellement après import :

Clic droit sur la VM → Configuration → Réseau → Avancé → Redirection de ports :

| Nom | Protocole | Port hôte | Port invité |
|---|---|---|---|
| SSH | TCP | 2222 | 22 |
| Kibana | TCP | 5601 | 5601 |

Connexion SSH : `ssh madeo@127.0.0.1 -p 2222`
Connexion Kibana : `http://localhost:5601`

---

## 3. Installation pas à pas

### 3.1 Elasticsearch + Kibana

```bash
# Installation via le dépôt officiel Elastic (suppose que le dépôt APT Elastic 8.x est déjà ajouté)
sudo apt update
sudo apt install -y elasticsearch kibana
sudo systemctl enable elasticsearch kibana
sudo systemctl start elasticsearch
sudo systemctl start kibana
```

Noter le mot de passe `elastic` généré automatiquement à l'installation. Si perdu :
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

**Connecter Kibana à Elasticsearch :**
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```
Copier le token affiché dans l'écran d'inscription de Kibana (`http://localhost:5601`), puis entrer le code de vérification :
```bash
sudo /usr/share/kibana/bin/kibana-verification-code
```

**Config `kibana.yml`** (pour un accès depuis l'hôte Windows) :
```yaml
server.host: "0.0.0.0"
```

### 3.2 syslog-ng → Elasticsearch

Installer le module HTTP (souvent déjà inclus) :
```bash
sudo apt install -y syslog-ng-mod-http
```

Créer `/etc/syslog-ng/conf.d/elasticsearch.conf` :
```
destination d_elasticsearch {
    http(
        url("https://localhost:9200/syslog-ng/_doc")
        method("POST")
        user("elastic")
        password("VOTRE_MOT_DE_PASSE")
        headers("Content-Type: application/json")
        body('{"@timestamp":"${ISODATE}","host":"${HOST}","program":"${PROGRAM}","message":"${MSGONLY}","facility":"${FACILITY}"}')
        tls(peer-verify(no))
    );
};

log {
    source(s_src);
    destination(d_elasticsearch);
};
```

Sécuriser le fichier (il contient un mot de passe en clair) :
```bash
sudo chmod 600 /etc/syslog-ng/conf.d/elasticsearch.conf
sudo chown root:root /etc/syslog-ng/conf.d/elasticsearch.conf
```

Valider et redémarrer :
```bash
sudo syslog-ng --syntax-only
sudo systemctl daemon-reload
sudo systemctl restart syslog-ng
```

Dans Kibana, créer une **Data View** : `syslog-ng*`, champ timestamp `@timestamp`.

### 3.3 Suricata (IDS)

```bash
sudo apt install -y suricata
```

**Config `/etc/suricata/suricata.yaml`** — deux sections à modifier :

```yaml
vars:
  address-groups:
    HOME_NET: "[10.0.2.0/24]"   # Adapter selon la plage réseau réelle de la VM
```

```yaml
af-packet:
  - interface: enp0s3            # Adapter selon le nom de l'interface (voir `ip addr show`)
```

**Télécharger les règles de détection (Emerging Threats Open) :**
```bash
sudo suricata-update
```

**Tester la configuration :**
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

**Démarrer le service :**
```bash
sudo systemctl enable suricata
sudo systemctl start suricata
```

### 3.4 Filebeat + module Suricata → Elasticsearch/Kibana

```bash
sudo apt install -y filebeat
sudo filebeat modules enable suricata
```

**Activer le module** dans `/etc/filebeat/modules.d/suricata.yml` :
```yaml
- module: suricata
  eve:
    enabled: true
```

**Configurer la sortie** dans `/etc/filebeat/filebeat.yml` :
```yaml
output.elasticsearch:
  ssl.verification_mode: none
  hosts: ["https://localhost:9200"]
  username: "elastic"
  password: "VOTRE_MOT_DE_PASSE"

setup.kibana:
  host: "localhost:5601"
```

Sécuriser :
```bash
sudo chmod 600 /etc/filebeat/filebeat.yml
sudo chown root:root /etc/filebeat/filebeat.yml
```

**Charger les index, pipelines et dashboards Kibana prédéfinis :**
```bash
sudo filebeat setup --pipelines --index-management \
  -E output.elasticsearch.hosts=['https://localhost:9200'] \
  -E output.elasticsearch.username=elastic \
  -E "output.elasticsearch.password=VOTRE_MOT_DE_PASSE" \
  -E output.elasticsearch.ssl.verification_mode=none

sudo filebeat setup --dashboards \
  -E output.elasticsearch.hosts=['https://localhost:9200'] \
  -E output.elasticsearch.username=elastic \
  -E "output.elasticsearch.password=VOTRE_MOT_DE_PASSE" \
  -E output.elasticsearch.ssl.verification_mode=none
```

**Démarrer le service :**
```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

## 4. Vérification de la chaîne complète

```bash
# Santé du cluster Elasticsearch
curl -k -u elastic:'VOTRE_MOT_DE_PASSE' https://localhost:9200/_cluster/health?pretty

# Vérifier les index créés
curl -k -u elastic:'VOTRE_MOT_DE_PASSE' https://localhost:9200/_cat/indices?v

# Générer une vraie alerte Suricata (site conçu pour les tests d'IDS, sans danger)
curl testmyids.com

# Vérifier que l'alerte a été loggée
sudo tail -5 /var/log/suricata/fast.log
```

Une alerte du type `GPL ATTACK_RESPONSE id check returned root` doit apparaître, puis être visible dans Kibana sous **Dashboards → [Filebeat Suricata] Alerts Overview** après quelques secondes.

---

## 5. Utilisation quotidienne

**Vérifier l'état de tous les services :**
```bash
sudo systemctl status elasticsearch kibana syslog-ng suricata filebeat
```

**Ordre de démarrage recommandé** (après un redémarrage de VM) : Elasticsearch peut prendre 1 à 4 minutes à démarrer selon la charge — attendre `active (running)` avant de tester Kibana ou de vérifier la santé du cluster.

**Accès Kibana** : `http://localhost:5601` (compte `elastic`)
**Discover** : logs bruts, filtrables par data view (`syslog-ng*`, `filebeat-*`)
**Dashboards** : rechercher "suricata" pour retrouver les dashboards prédéfinis (Events Overview, Alerts Overview)

---

## 6. Créer un scénario d'intrusion

Chaque scénario doit suivre cette structure pour rester cohérent dans la documentation finale :

1. **Objectif** : quelle technique MITRE ATT&CK ou quel type d'attaque est simulé
2. **Commande(s) exécutée(s)** sur la VM ou depuis une machine externe ciblant la VM
3. **Alerte(s) attendue(s)** : quelle(s) règle(s) Suricata devrai(en)t se déclencher
4. **Capture d'écran** du dashboard Kibana montrant l'alerte détectée
5. **Analyse** : pourquoi cette activité est considérée malveillante, quel serait le comportement en environnement réel

### Exemples de scénarios possibles

| # | Scénario | Commande de test (exemple) |
|---|---|---|
| 1 | Scan de ports (reconnaissance) | `nmap -sS 10.0.2.15` depuis une autre machine |
| 2 | Tentative de brute-force SSH | `hydra -l root -P wordlist.txt ssh://10.0.2.15` |
| 3 | Exécution de commande suspecte via HTTP | `curl testmyids.com` (déjà validé) |
| 4 | Téléchargement de fichier exécutable suspect | Télécharger un binaire depuis un serveur de test EICAR |
| 5 | Trafic vers un domaine malveillant connu | Requête DNS vers un domaine présent dans les règles ET Open |

Pour trouver le nom exact d'une règle déclenchée et l'inclure dans le rapport :
```bash
sudo grep -i "<mot-clé>" /var/lib/suricata/rules/suricata.rules
```

---

## 7. Dépannage courant

| Symptôme | Cause probable | Solution |
|---|---|---|
| `elasticsearch.service` reste en `activating (start)` longtemps | RAM insuffisante, démarrage lent normal | Attendre jusqu'à 4-5 min ; vérifier `free -h` |
| `kernel OOM killer killed some processes` dans `journalctl -u elasticsearch` | Manque de RAM sur la VM | Augmenter la RAM allouée à la VM (5-6 Go recommandé) |
| `chown: invalid group: 'root:syslog-ng'` | Le groupe n'existe pas sous ce nom sur cette distro | Utiliser `chmod 600` + `chown root:root` à la place |
| `Warning: The unit file... changed on disk` | systemd n'a pas rechargé sa vue des unités | `sudo systemctl daemon-reload` avant de redémarrer le service |
| `Package 'libpcre3-dev' has no installation candidate` | Paquet obsolète sur Ubuntu 26.04 | Utiliser `libpcre2-dev` à la place |
| `Unable to locate package snort` | Snort retiré des dépôts Ubuntu 26.04 | Utiliser Suricata (voir section 3.3) |
| Warning "No rule files match the pattern" au démarrage de Suricata | Règles pas encore téléchargées | `sudo suricata-update` |

---

## 8. Identifiants de démonstration

> ⚠️ À ne jamais publier tels quels dans un dépôt GitHub public. Les remplacer par `<VOTRE_MOT_DE_PASSE>` dans toute version partagée du code, et transmettre les vraies valeurs uniquement par un canal privé (message direct, gestionnaire de mots de passe partagé).

- SSH : `madeo@127.0.0.1 -p 2222`
- Kibana / Elasticsearch : utilisateur `elastic`

---

## 9. Répartition des tâches (groupe de 4)

| Membre | Responsabilité |
|---|---|
| Guy (lead) | Déploiement et intégration complète (ELK, syslog-ng, Suricata, Filebeat) |
| Coéquipier 2 | Validation/documentation de la substitution Snort → Suricata |
| Coéquipier 3 | Scénarios d'intrusion 1-2, mise en place des alertes email/Slack |
| Coéquipier 4 | Scénarios d'intrusion 3-5, dashboards Kibana |

---

## 10. Prochaines étapes

- [ ] Concevoir et exécuter les 5 scénarios d'intrusion
- [ ] Mettre en place les alertes email/Slack sur détection
- [ ] Finaliser la documentation GitHub (captures d'écran, analyse par scénario)
