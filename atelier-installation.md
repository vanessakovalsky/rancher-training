# Atelier RKE2 - Installation d'un environnement Kubernetes complet

## Objectifs pédagogiques
- Comprendre l'architecture RKE2
- Installer et configurer un cluster RKE2 multi-nœuds
- Déployer des applications de test
- Maîtriser les outils de gestion kubectl et crictl

## Prérequis techniques


### Prérequis

- **Connectivité réseau** entre toutes les machines
- **Accès root ou sudo** sur toutes les machines
- **Ports réseau ouverts** (voir section dédiée)

### Outils nécessaires
- SSH client
- Éditeur de texte (nano, vim)
- Navigateur web (pour l'interface de gestion optionnelle)


## Phase 1 : Préparation de l'environnement

### Etape 1.0 : Création des machines

* En environnement Linux, on va installer multipass et créer une VM :
```
sudo snap install multipass
multipass launch --name=rke-master
```
* Ouvrir l'utilitaire multipass depuis le menu et vous connecter à l'instance rke-master que l'on vient de créer pour la suite

### Étape 1.1 : Configuration réseau et hostnames

**Sur toutes les machines :**

```bash
# 1. Définir les hostnames (adapter les IP selon votre environnement)
sudo hostnamectl set-hostname rke2-master  # Sur le master
# 2. Configurer le fichier hosts
sudo nano /etc/hosts
```

**Ajouter dans /etc/hosts (sur toutes les machines) :**
```
192.168.1.10 rke2-master
```

### Étape 1 : Ouverture des ports firewall

**Sur le nœud master :**
```bash
# Pour Ubuntu/Debian
sudo ufw allow 6443/tcp    # Kubernetes API
sudo ufw allow 10250/tcp   # Kubelet API
sudo ufw allow 2379:2380/tcp # etcd
sudo ufw allow 9345/tcp    # RKE2 supervisor API

# Pour CentOS/RHEL
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=9345/tcp
sudo firewall-cmd --reload
```


### Étape 1.3 : Désactivation du swap

**Sur toutes les machines :**
```bash
# Désactiver le swap temporairement
sudo swapoff -a

# Désactiver le swap de façon permanente
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Vérifier que le swap est désactivé
free -h
```

### Étape 1.4 : Configuration des modules kernel

**Sur toutes les machines :**
```bash
# Créer le fichier de configuration des modules
sudo tee /etc/modules-load.d/rke2.conf <<EOF
overlay
br_netfilter
EOF

# Charger les modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configuration sysctl pour Kubernetes
sudo tee /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Appliquer les paramètres sysctl
sudo sysctl --system
```

---

## Phase 2 : Installation du nœud master RKE2

### Étape 2.1 : Installation de RKE2 sur le master

**Sur le nœud master uniquement :**

```bash
# 1. Télécharger le script d'installation RKE2
sudo curl -sfL https://get.rke2.io | sudo sh -

# 2. Créer le répertoire de configuration
sudo mkdir -p /etc/rancher/rke2/

# 3. Créer le fichier de configuration
sudo tee /etc/rancher/rke2/config.yaml <<EOF
# Configuration du serveur RKE2
write-kubeconfig-mode: "0644"
tls-san:
  - "rke2-master"
  - "192.168.1.10"  # IP du master
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
EOF
```

### Étape 2.2 : Démarrage du service RKE2

```bash
# 1. Activer et démarrer le service RKE2
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

# 2. Vérifier le statut du service
sudo systemctl status rke2-server.service

# 3. Suivre les logs de démarrage (optionnel)
sudo journalctl -u rke2-server -f
```

**⚠️ Point de vérification :** Le service doit être en état "active (running)"

### Étape 2.3 : Configuration de kubectl

```bash
# 1. Ajouter les binaires RKE2 au PATH
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
source ~/.bashrc

# 2. Configurer kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 3. Tester kubectl
kubectl get nodes
kubectl get pods -A
```

**Résultat attendu :** Le nœud master doit apparaître en état "Ready"

### Étape 2.4 : Récupération du token de jointure

```bash
# Afficher le token nécessaire pour les workers
sudo cat /var/lib/rancher/rke2/server/node-token
```

**📝 Important :** Noter ce token, il sera nécessaire pour joindre les workers au cluster

---

## Phase 3 (optionnel non utile pour la formation) : Ajout des nœuds worker

### Étape 3.1 : Installation de RKE2 sur les workers

**Sur chaque nœud worker :**

```bash
# 1. Télécharger le script d'installation RKE2
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

# 2. Créer le répertoire de configuration
sudo mkdir -p /etc/rancher/rke2/

# 3. Créer le fichier de configuration
sudo tee /etc/rancher/rke2/config.yaml <<EOF
server: https://192.168.1.10:9345  # IP du master
token: K10... # Token récupéré du master (remplacer par le vrai token)
EOF
```

### Étape 3.2 : Démarrage des agents RKE2

**Sur chaque worker :**

```bash
# 1. Activer et démarrer le service agent
sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service

# 2. Vérifier le statut
sudo systemctl status rke2-agent.service

# 3. Suivre les logs
sudo journalctl -u rke2-agent -f
```

### Étape 3.3 : Vérification du cluster

**Depuis le nœud master :**

```bash
# Vérifier que tous les nœuds sont présents
kubectl get nodes -o wide

# Vérifier l'état des pods système
kubectl get pods -n kube-system

# Afficher les informations détaillées du cluster
kubectl cluster-info
```

**Résultat attendu :** Tous les nœuds doivent être en état "Ready"


## Phase 4 : Tests et validation

### Étape 4.1 : Déploiement d'une application de test

```bash
# 1. Créer un namespace de test
kubectl create namespace test-app

# 2. Déployer une application nginx
kubectl create deployment nginx --image=nginx:latest -n test-app

# 3. Exposer l'application
kubectl expose deployment nginx --port=80 --type=NodePort -n test-app

# 4. Vérifier le déploiement
kubectl get all -n test-app
```

### Étape 4.2 : Test de connectivité

```bash
# 1. Obtenir le port NodePort
kubectl get svc nginx -n test-app

# 2. Tester l'accès depuis l'extérieur
curl http://192.168.1.11:NODEPORT  # Remplacer NODEPORT par le port affiché
```

### Étape 4.3 : Test de scalabilité

```bash
# 1. Scaler l'application
kubectl scale deployment nginx --replicas=3 -n test-app

# 2. Vérifier la répartition des pods
kubectl get pods -n test-app -o wide

# 3. Tester la résistance (supprimer un pod)
kubectl delete pod <nom-du-pod> -n test-app
kubectl get pods -n test-app
```

---

## Phase 5 : Surveillance et maintenance

### Étape 5.1 : Commandes de diagnostic essentielles

```bash
# État du cluster
kubectl get nodes
kubectl get pods --all-namespaces

# Logs des composants système
sudo journalctl -u rke2-server  # Sur le master
sudo journalctl -u rke2-agent   # Sur les workers

# Utilisation des ressources
kubectl top nodes
kubectl top pods --all-namespaces

# Événements du cluster
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Dépannage courant

### Problème : Les nœuds ne rejoignent pas le cluster
**Solutions :**
- Vérifier la connectivité réseau (ping entre les nœuds)
- Contrôler que les ports sont ouverts (telnet ip port)
- Vérifier le token de jointure
- Examiner les logs : `sudo journalctl -u rke2-agent -f`

### Problème : Les pods restent en état Pending
**Solutions :**
- Vérifier les ressources disponibles : `kubectl describe node`
- Contrôler les taints et tolerations
- Examiner les événements : `kubectl describe pod <nom-pod>`

### Problème : Problèmes de réseau entre pods
**Solutions :**
- Vérifier la configuration CNI
- Tester la résolution DNS : `kubectl exec -it <pod> -- nslookup kubernetes.default`
- Contrôler les NetworkPolicies

---

## Ressources complémentaires

- **Documentation officielle RKE2 :** https://docs.rke2.io/
- **Kubernetes Documentation :** https://kubernetes.io/docs/
- **Best practices :** https://kubernetes.io/docs/concepts/cluster-administration/
- **Troubleshooting guide :** https://kubernetes.io/docs/tasks/debug-application-cluster/

---

## Validation des acquis

À la fin de cet atelier, les stagiaires doivent être capables de :
- ✅ Installer un cluster RKE2 fonctionnel
- ✅ Configurer kubectl et crictl
- ✅ Déployer et gérer des applications
- ✅ Diagnostiquer les problèmes courants
- ✅ Effectuer des opérations de maintenance de base
