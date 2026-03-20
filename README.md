# 🛡️ SOC Lab Conteneurisé — L2 SIMAC 2025-2026

**Auteur :** Mouhamed Moustapha Sabaly - Penda Sall Pouye  
**Module :** Cloud et Infrastructures Virtuelles  
**Enseignant :** Dr GUEYE  
**Formation :** L2 SIMAC P11  
**Date :** 19 Mars 2026

---

## 📋 Description

Déploiement d'un **Security Operations Center (SOC)** complet en utilisant Docker Compose.  
La stack surveille, détecte et répond aux incidents de sécurité en temps réel.

| Service | Version | Rôle | Port |
|---|---|---|---|
| Wazuh Indexer | 4.14.3 | Stockage alertes (OpenSearch) | 9200 |
| Wazuh Manager | 4.14.3 | SIEM + Filebeat | 55000, 1514 |
| Wazuh Dashboard | 4.14.3 | Interface web SIEM | 443 |
| Suricata | latest | IDS/IPS réseau | host network |
| TheHive | 5.4 | Gestion incidents (IRP) | 9000 |
| Grafana | 10.2.0 | Visualisation métriques | 3000 |
| Cassandra | 4.1 | Base de données TheHive | 9042 |
| DVWA | 1.10 | Cible vulnérable (tests) | 8080 |

---

## 🗂️ Structure du dépôt

```
soc-lab-Mouhamed-Penda/
├── README.md                        ← Ce fichier
├── docker-compose.yml               ← Stack SOC complète (8 services)
├── .env.example                     ← Variables fictives — JAMAIS le vrai .env
├── .gitignore                       ← .env et certificats ignorés
├── deploy_soc_lab_partie1.sh        ← Script déploiement automatisé (Partie 1)
├── deploy_soc_lab_partie2.sh        ← Script attaques automatisées (Partie 2)
├── wazuh/
│   └── config/
│       ├── wazuh_cluster/
│       │   └── wazuh_manager.conf   ← Configuration ossec manager
│       ├── wazuh_dashboard/
│       │   ├── opensearch_dashboards.yml
│       │   └── wazuh.yml            ← Connexion API (run_as: false)
│       ├── wazuh_indexer/
│       │   ├── wazuh.indexer.yml
│       │   └── internal_users.yml
│       └── wazuh_indexer_ssl_certs/ ← 11 certificats TLS (.gitignored)
├── suricata/
│   ├── rules/                       ← Règles Suricata personnalisées
│   └── logs/                        ← eve.json, fast.log (.gitkeep)
├── grafana/
│   └── dashboards/                  ← Dashboards JSON exportés
├── thehive/
│   └── config/                      ← Configuration TheHive
└── rapport/
    |-- rapport SOC LAB CONTENEURISE Guide complet de deploiement et simulation d'attaques.pdf
    ├── rapport-final.pdf            ← Rapport technique (16 pages)
    └── captures/                    ← Preuves des attaques
        ├── nmap_scan.txt            ← Résultats scan Nmap
        ├── hydra_ssh.txt            ← Log attaque Hydra SSH
        ├── sqlmap_scan.txt          ← Résultats SQL Injection
        └── resume_attaques.txt      ← Résumé des 3 attaques
```

---

## ⚡ Déploiement rapide

### Prérequis

- Ubuntu 24.04 LTS (ou équivalent Debian/Ubuntu)
- Minimum **6 Go RAM** (7.7 Go recommandés)
- Docker 24+ et Docker Compose v2
- Accès sudo

### Option A — Script automatisé (recommandé)

```bash
# Partie 1 : Infrastructure Docker
chmod +x deploy_soc_lab_partie1.sh
./deploy_soc_lab_partie1.sh

# Partie 2 : Simulation d'attaques
chmod +x deploy_soc_lab_partie2.sh
./deploy_soc_lab_partie2.sh
```

### Option B — Déploiement manuel étape par étape

**1. Cloner le repo et configurer les variables**
```bash
git clone <URL_DEPOT> soc-lab-Mouhamed-Penda
cd soc-lab-Mouhamed-Penda
cp .env.example .env
nano .env   # Remplir les vrais mots de passe
```

**2. Configurer vm.max_map_count (obligatoire pour OpenSearch)**
```bash
# Temporaire (perdu au redémarrage)
sudo sysctl -w vm.max_map_count=262144

# Permanent (survit aux redémarrages)
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```
> **Pourquoi ?** OpenSearch utilise des fichiers mappés en mémoire (mmap) pour ses index.
> La valeur par défaut Linux (65 530) est insuffisante et provoque une erreur au démarrage.

**3. Installer Docker Compose v2**
```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL \
  https://github.com/docker/compose/releases/download/v2.24.5/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

**4. Générer les certificats TLS**
```bash
# Créer certs.yml avec les 3 nœuds
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
cp wazuh/config/certs.yml config.yml
sudo bash wazuh-certs-tool.sh -A
sudo mv wazuh-certificates/* wazuh/config/wazuh_indexer_ssl_certs/
sudo chown -R $USER:$USER wazuh/config/wazuh_indexer_ssl_certs/
cp wazuh/config/wazuh_indexer_ssl_certs/root-ca.pem \
   wazuh/config/wazuh_indexer_ssl_certs/root-ca-manager.pem
sudo rm -rf wazuh-certificates/ wazuh-certs-tool.sh config.yml
```

**5. Lancer la stack**
```bash
# Valider le YAML
docker compose config --quiet && echo "✅ YAML valide"

# Lancer tous les services
docker compose up -d

# Vérifier l'état
docker compose ps
```

**6. Appliquer les fixes post-démarrage**

> ⚠️ Ces fixes sont nécessaires après chaque recréation du container `wazuh.manager`

```bash
# Fix mot de passe wazuh-wui (ERROR3099)
docker exec soc-lab-mouhamed-penda-wazuh.manager-1 \
  /var/ossec/framework/python/bin/python3 -c "
import sqlite3, sys
sys.path.insert(0, '/var/ossec/framework/python/lib/python3.10/site-packages')
from werkzeug.security import generate_password_hash
hashed = generate_password_hash('MyS3cr37P450r.*-')
db = '/var/ossec/api/configuration/security/rbac.db'
conn = sqlite3.connect(db); cur = conn.cursor()
cur.execute('UPDATE users SET password = ? WHERE username = ?', (hashed, 'wazuh-wui'))
conn.commit(); print('Done:', cur.rowcount); conn.close()
"

# Fix credentials Filebeat (SANS pkill)
docker exec soc-lab-mouhamed-penda-wazuh.manager-1 bash -c "
  sed -i 's|  #username:|  username: admin|' /etc/filebeat/filebeat.yml
  sed -i 's|  #password:|  password: SecretPassword|' /etc/filebeat/filebeat.yml
"

# Redémarrer le manager pour appliquer les changements
docker compose restart wazuh.manager
```

---

## 🌐 Accès aux interfaces

| Interface | URL | Login | Mot de passe |
|---|---|---|---|
| 🔐 Wazuh Dashboard | https://localhost | admin | SecretPassword |
| 📊 Grafana | http://localhost:3000 | admin | GrafanaAdmin123! |
| 🐝 TheHive | http://localhost:9000 | admin@thehive.local | secret |
| 🎯 DVWA | http://localhost:8080 | admin | password |
| 🔎 OpenSearch API | https://localhost:9200 | admin | SecretPassword |

> **Note :** Le certificat Wazuh Dashboard est auto-signé — accepter l'exception de sécurité dans le navigateur.

---

## ⚔️ Simulation des attaques (Partie 2)

> ⛔ **Usage légal uniquement** — Toutes les attaques sont effectuées sur vos propres containers dans un environnement isolé.

### Attaque 1 — Reconnaissance Nmap (TA0043)

```bash
nmap -sV -sC -A localhost 2>&1 | tee rapport/captures/nmap_scan.txt
```

**Résultat attendu :** 7 ports découverts (22, 443, 631, 3000, 8080, 9000, 9200)

### Attaque 2 — SSH Brute Force Hydra (T1110.001)

```bash
echo -e "password\n123456\nroot\nadmin\npassword123\ntoor\nqwerty\nletmein\n12345" \
  > /tmp/passwords.txt

hydra -l root -P /tmp/passwords.txt ssh://localhost -t 4 -V \
  2>&1 | tee rapport/captures/hydra_ssh.txt
```

**Résultat attendu :** 14 alertes dans Wazuh Threat Hunting (T1110.001 détecté)

### Attaque 3 — SQL Injection DVWA (T1190)

Ouvrir `http://localhost:8080/vulnerabilities/sqli/` et entrer :

```sql
-- Payload 1 : exfiltration des utilisateurs
1' OR '1'='1

-- Payload 2 : exfiltration des hachages MD5
%' AND 1=0 UNION SELECT null, concat(user,0x3a,password) FROM users#
```

**Résultat attendu :** 5 hachages MD5 exfiltrés et crackés

---

## 🔧 Commandes utiles

```bash
# État des services
docker compose ps

# Logs d'un service spécifique
docker compose logs -f wazuh.manager
docker compose logs -f wazuh.dashboard

# Redémarrer un service
docker compose restart wazuh.manager

# Vérifier les alertes SSH dans les logs Wazuh
docker exec soc-lab-mouhamed-penda-wazuh.manager-1 \
  grep -c "sshd" /var/ossec/logs/alerts/alerts.log

# Vérifier les indices OpenSearch
curl -sk -u admin:SecretPassword \
  https://localhost:9200/_cat/indices/wazuh-alerts* 2>/dev/null

# Vérifier l'API Wazuh
curl -sk -u wazuh-wui:'MyS3cr37P450r.*-' \
  https://localhost:55000/security/user/authenticate -X GET | head -c 100

# Arrêter la stack
docker compose down

# Arrêter ET supprimer les volumes (reset complet)
docker compose down -v
```

---

## 🐛 Problèmes connus et solutions

| Erreur | Cause | Solution |
|---|---|---|
| `YAML invalide ligne 32` | healthcheck format compact | Séparer en 4 lignes distinctes |
| `Cassandra exited(1)` | MAX_HEAP_SIZE + G1GC incompatibles | Supprimer MAX_HEAP_SIZE et HEAP_NEWSIZE |
| `ENOENT opensearch_dashboards.yml` | Volumes manquants dans dashboard | Ajouter les 5 volumes obligatoires |
| `ERROR3099 Invalid credentials` | Hash werkzeug incorrect dans rbac.db | Régénérer le hash via Python Wazuh |
| `#username commenté dans filebeat.yml` | Config par défaut Wazuh | `sed -i` pour décommenter (sans pkill) |
| `unable to verify the first certificate` | Anciens certs en mémoire indexer | `docker compose rm -sf wazuh.indexer` |
| `wazuh-alerts* index vide` | Filebeat sans credentials | Appliquer le fix Filebeat + restart |

---

## 🏗️ Architecture réseau

```
┌─────────────────── HÔTE UBUNTU 24.04 ───────────────────────┐
│                                                               │
│  ┌────── soc-frontend (172.19.0.0/16) ─────────────────┐    │
│  │  Wazuh Dashboard :443  │  Grafana :3000              │    │
│  │  TheHive :9000         │  DVWA :8080                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌────── soc-backend (172.18.0.0/16) ──────────────────┐    │
│  │  Wazuh Indexer :9200  │  Wazuh Manager :55000        │    │
│  │  Cassandra :9042      │  Suricata (host network)     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  Note: Wazuh Dashboard est sur les DEUX réseaux (obligatoire)│
└───────────────────────────────────────────────────────────────┘
```

---

## 📊 Résultats des attaques

| Attaque | MITRE | Alertes | Résultat |
|---|---|---|---|
| Nmap Reconnaissance | TA0043 | — | 7 ports découverts |
| SSH Brute Force | T1110.001 | 14 alertes Wazuh | 0 accès obtenu ✅ |
| SQL Injection DVWA | T1190 | — | 5 hachages MD5 exfiltrés |

---

## 📄 Livrables

- `rapport/rapport-final.pdf` — Rapport technique 16 pages
- `deploy_soc_lab_partie1.sh` — Script déploiement automatisé
- `deploy_soc_lab_partie2.sh` — Script attaques automatisées
- `rapport/captures/` — Preuves des 3 attaques

---

## ⚠️ Sécurité

- Le fichier `.env` contient les vrais mots de passe — **ne jamais committer**
- Les certificats TLS sont auto-signés — à remplacer par Let's Encrypt en production
- DVWA est volontairement vulnérable — **ne jamais exposer sur un réseau public**

---

*Projet réalisé dans le cadre du module Cloud et Infrastructures Virtuelles — L2 SIMAC 2025-2026*
