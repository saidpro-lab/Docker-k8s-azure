# Docker-k8s-azure
Déploiement de GLPI sur Azure Kubernetes Service (AKS)
# 🐳 Docker Swarm — Déploiement HA de GLPI

> TP pratique — Formation AIS Bac+3 | Module : Infrastructure & Conteneurisation  
> Durée : 4h | Prérequis : TP Docker Compose v1 réalisé

---

## 🎯 Objectif

Déployer GLPI en **haute disponibilité** sur un cluster Docker Swarm multi-nœuds avec stockage partagé NFS, puis tester la résilience face à une panne simulée.

---

## 🖥️ Infrastructure du lab

| VM | Rôle | IP Fixe | RAM | CPU |
|----|------|---------|-----|-----|
| SRV-Manager | Swarm Manager | 192.168.169.10 | 2 GB | 4 |
| SRV-Worker | Swarm Worker | 192.168.169.20 | 2 GB | 2 |
| SRV-STORAGE | NFS Server | 192.168.169.30 | 2 GB | 2 |

> Les 3 VMs Debian 12 sont fournies via fichier OVF à importer dans VMware Workstation.  
> Réseau : VMware NAT (toutes les VMs sur le même sous-réseau).

---

## 📦 Stack déployée

| Service | Image | Replicas | Port |
|---------|-------|----------|------|
| GLPI | `saidformation59/labs:glpi-app` | 2 | 8080 |
| MariaDB | `mysql:8.0` | 1 | — |
| Redis | `redis:7-alpine` | 1 | — |

---

## 🚀 Démarrage rapide

### 1. Cloner le repo sur SRV-Manager

```bash
git clone https://github.com/saidpro-lab/Swarm-v2.git
cd Swarm-v2
git checkout swarm-v2
```

### 2. Configurer les variables d'environnement

```bash
cp .env.example .env
nano .env
```

```env
MYSQL_ROOT_PASSWORD=VotreMotDePasse!
MYSQL_DATABASE=glpi
MYSQL_USER=glpi_user
MYSQL_PASSWORD=GlpiPassword2026!
GLPI_DB_HOST=db
GLPI_DB_NAME=glpi
GLPI_PORT=8080
```

```bash
export $(cat .env | xargs)
```

### 3. Initialiser le cluster Swarm

Sur **SRV-Manager** :
```bash
docker swarm init --advertise-addr 192.168.169.10
```

Sur **SRV-Worker** (avec le token généré ci-dessus) :
```bash
docker swarm join --token <TOKEN> 192.168.169.10:2377
```

### 4. Déployer la stack

```bash
docker stack deploy -c docker-stack.yml glpi
```

### 5. Vérifier

```bash
docker service ls
docker service ps glpi_glpi
watch docker service ls
```

### 6. Accéder à GLPI

```
http://192.168.169.10:8080
http://192.168.169.20:8080
```

---

## 🗂️ Structure du repo

```
Swarm-v2/
├── docker-stack.yml       # Stack Swarm (services, réseaux, volumes)
├── .env.example           # Template des variables d'environnement
└── README.md
```

---

## 🔧 Commandes utiles

```bash
docker node ls                        # Lister les nœuds du cluster
docker service ls                     # Lister les services
docker service ps glpi_glpi           # Voir les tasks d'un service
docker service scale glpi_glpi=3      # Scaler un service
docker service logs glpi_glpi         # Voir les logs
docker service update --force glpi_glpi  # Forcer la redistribution des replicas
docker stack rm glpi                  # Supprimer la stack
```

---

## 🔌 Ports Swarm à ouvrir

| Port | Protocole | Usage |
|------|-----------|-------|
| 2377 | TCP | Communication Manager ↔ Worker |
| 7946 | TCP + UDP | Découverte entre nœuds |
| 4789 | UDP | Réseau Overlay VXLAN |

---

## 💾 Stockage NFS (SRV-STORAGE)

Les données persistantes sont partagées via NFS depuis `SRV-STORAGE` :

| Partage NFS | Point de montage | Contenu |
|-------------|-----------------|---------|
| `/exports/glpi-files` | `/mnt/glpi-files` | Fichiers GLPI |
| `/exports/glpi-db` | `/mnt/glpi-db` | Base de données |
| `/exports/glpi-redis` | `/mnt/glpi-redis` | Cache Redis |

---

## 📋 Concepts abordés

- Docker Swarm : Manager, Worker, Service, Task, Stack
- Réseau Overlay et encapsulation VXLAN
- Stockage distribué NFS avec `_netdev` dans `/etc/fstab`
- Haute disponibilité et auto-healing
- Simulation de panne et observation du comportement Swarm
- Scaling dynamique (`docker service scale`)

---

## 🏫 Formation

**AIS Bac+3** — Administrateur d'Infrastructures Sécurisées  
Module : Infrastructure & Conteneurisation  
Repo lab associé : [saidpro-lab/Swarm-v2](https://github.com/saidpro-lab/Swarm-v2)
