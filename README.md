# Docker-k8s-azure
Déploiement de GLPI sur Azure Kubernetes Service (AKS)

# ☸️ Kubernetes AKS — Déploiement de GLPI sur Azure

> TP pratique — Formation AIS Bac+3 | Module : Infrastructure Cloud & Conteneurisation  
> Prérequis : Docker Compose, Docker Swarm, bases Linux, YAML  
> Outils : Azure CLI, kubectl, PowerShell 7, VS Code

---

## 🎯 Objectif

Déployer GLPI sur **Azure Kubernetes Service (AKS)** en deux versions :

- **V1** — Déploiement de base avec IP publique (HTTP)
- **V2** — Sécurisation HTTPS avec Ingress-nginx + cert-manager + Let's Encrypt automatique

---

## 📐 Architecture

### V1 — GLPI de base (HTTP)

```
INTERNET
    │ HTTP :80
    ▼
Azure Load Balancer (IP publique auto)
    │
Cluster AKS (Belgium Central)
    └── Namespace: glpi
        ├── Pod GLPI (diouxx/glpi)   ──► Service: LoadBalancer (exposé Internet)
        └── Pod MariaDB (10.11)      ──► Service: ClusterIP (interne)
                                              └── PVC 5Gi → Azure Disk (auto)
```

Objets K8s créés : `Namespace` · `Secret` · `ConfigMap` · `PVC` · `2x Deployment` · `2x Service`

### V2 — HTTPS avec Ingress (architecture finale)

```
INTERNET
    │ HTTPS :443  →  https://glpi.<IP>.nip.io
    ▼
Azure Load Balancer (ingress-nginx)
    │
Cluster AKS
    ├── Namespace: ingress-nginx
    │   └── Pod ingress-nginx-controller (terminaison SSL + routing)
    ├── Namespace: cert-manager
    │   └── cert-manager + ClusterIssuer (Let's Encrypt automatique)
    └── Namespace: glpi
        ├── Pod GLPI          ──► Service: ClusterIP
        └── Pod MariaDB       ──► Service: ClusterIP
                                        └── PVC 5Gi → Azure Disk
```

Objets ajoutés en V2 : `ingress-nginx` · `cert-manager` · `ClusterIssuer` · `Ingress`

---

## ⚖️ Comparatif des approches

| Critère | Docker Compose | Docker Swarm | Kubernetes AKS |
|---------|---------------|-------------|----------------|
| Infrastructure | 1 seule VM | 3 VMs manuelles | VMs gérées par Azure |
| Haute dispo | ❌ | ⚠️ Partielle | ✅ Native |
| Stockage | Local | NFS manuel | Azure Disk auto |
| SSL | Manuel | Manuel | cert-manager auto |
| IP publique | VMware fixe | VMware NAT | Azure auto |

---

## 🚀 Démarrage rapide

### Prérequis

```powershell
winget install Microsoft.AzureCLI
az aks install-cli
winget install Microsoft.PowerShell
```

### 1. Connexion Azure

```bash
az login --tenant <TENANT_ID>
az account set --subscription <SUBSCRIPTION_ID>
```

### 2. Créer le cluster AKS (portail Azure)

| Paramètre | Valeur |
|-----------|--------|
| Resource Group | `rg-gX-glpi` |
| Nom du cluster | `k8s-cluster` |
| Configuration | Dev/Test |
| Taille VM | Standard_B2s |
| Nœuds | 2 |
| Zones de dispo | Aucune |

### 3. Connexion au cluster

```bash
az aks get-credentials --resource-group rg-gX-glpi --name k8s-cluster
kubectl get nodes
```

### 4. Déployer la stack (V1)

```bash
kubectl create namespace glpi
kubectl apply -f manifests/
kubectl get pods -n glpi -w
```

### 5. Récupérer l'IP publique

```bash
kubectl get service glpi -n glpi
# → accéder à http://<EXTERNAL-IP>
```

**Configuration assistant GLPI :**
- Serveur SQL : `mariadb`
- Utilisateur : `glpi_user`
- Mot de passe : `GlpiP@ssw0rd2026!`
- Base de données : `glpi`

---

## 🔒 Partie 2 — Activation HTTPS

```bash
# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl get pods -n cert-manager -w

# ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
kubectl get service ingress-nginx-controller -n ingress-nginx
# → noter l'EXTERNAL-IP

# Éditer 06-ingress.yaml et 07-clusterissuer.yaml avec votre IP, puis :
kubectl apply -f manifests/
kubectl get certificate -n glpi -w
# → attendre READY = True
```

Accès : `https://glpi.<IP>.nip.io`

---

## 🗂️ Structure du repo

```
Docker-k8s-azure/
├── manifests/
│   ├── 00-namespace.yaml
│   ├── 01-secret.yaml
│   ├── 02-configmap.yaml
│   ├── 03-pvc.yaml
│   ├── 04-mariadb.yaml
│   ├── 05-glpi.yaml
│   ├── 06-ingress.yaml
│   └── 07-clusterissuer.yaml
└── README.md
```

---

## 🔧 Commandes utiles

```bash
kubectl get pods -n glpi -w                     # Surveillance temps réel
kubectl logs <pod-name> -n glpi                 # Logs d'un pod
kubectl describe pod <pod-name> -n glpi         # Debug
kubectl get pvc -n glpi                         # Volumes persistants
kubectl get certificate -n glpi                 # État certificat SSL
kubectl get ingress -n glpi                     # Règles ingress
```

---

## ⚠️ Nettoyage obligatoire

> Un cluster AKS qui tourne coûte de l'argent. Supprimez les ressources à la fin du TP !

```bash
az group delete --name rg-gX-glpi --yes --no-wait
az group delete --name NetworkWatcherRG --yes --no-wait
az group list --output table
```

---

## 🏫 Formation

**AIS Bac+3** — Administrateur d'Infrastructures Sécurisées  
Module : Infrastructure Cloud & Conteneurisation
