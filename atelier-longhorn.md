# Atelier Longhorn : Installation et Configuration
## Durée : 30 minutes

### Objectifs pédagogiques
À la fin de cet atelier, vous serez capable de :
- Comprendre les concepts de base de Longhorn
- Installer Longhorn sur un cluster RKE2 via Rancher
- Configurer le stockage persistant avec Longhorn
- Créer et gérer des volumes persistants
- Effectuer des opérations de sauvegarde et restauration

---

## Partie 1 : Introduction à Longhorn (5 minutes)

### Qu'est-ce que Longhorn ?
Longhorn est une solution de stockage distribué cloud-native pour Kubernetes qui fournit :
- **Stockage persistant** : Volumes répliqués et hautement disponibles
- **Sauvegarde et restauration** : Snapshots et backups vers stockage externe
- **Réplication** : Données répliquées sur plusieurs nœuds
- **Interface graphique** : Dashboard pour la gestion

### Architecture Longhorn
- **Longhorn Manager** : Orchestrateur principal
- **Longhorn Engine** : Moteur de stockage par volume
- **Instance Manager** : Gestionnaire des processus
- **CSI Driver** : Interface Container Storage Interface

---

## Partie 2 : Prérequis et vérifications (5 minutes)

### Vérifications préalables

#### 1. Vérifier les nœuds du cluster
```bash
kubectl get nodes -o wide
```

#### 2. Vérifier les prérequis système
```bash
# Vérifier que open-iscsi est installé sur chaque nœud
kubectl get nodes -o jsonpath='{.items[*].metadata.name}' | xargs -I {} kubectl debug node/{} -it --image=busybox -- chroot /host sh -c 'which iscsiadm'
```

#### 3. Vérifier l'espace disque disponible
```bash
# Vérifier l'espace sur /var/lib/longhorn
kubectl get nodes -o jsonpath='{.items[*].metadata.name}' | xargs -I {} kubectl debug node/{} -it --image=busybox -- chroot /host df -h /var/lib/longhorn
```

#### 4. Installer les dépendances manquantes (si nécessaire)
```bash
# Sur chaque nœud (exemple Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y open-iscsi util-linux

# Sur open suse

sudo zypper install open-iscsi util-linux
```

---

## Partie 3 : Installation via Rancher (10 minutes)

### Méthode 1 : Installation via l'interface Rancher

#### Étape 1 : Accéder au catalogue d'applications
1. Connectez-vous à l'interface Rancher
2. Sélectionnez votre cluster RKE2
3. Allez dans **Apps & Marketplace** → **Charts**
4. Recherchez "Longhorn"

#### Étape 2 : Configuration de l'installation
```yaml
# Paramètres de configuration recommandés
defaultSettings:
  defaultDataPath: "/var/lib/longhorn"
  defaultReplicaCount: 3
  createDefaultDiskLabeledNodes: true
  defaultLonghornStaticStorageClass: "longhorn"
  backupstorePollInterval: "300"

persistence:
  defaultClass: true
  defaultClassReplicaCount: 3
  reclaimPolicy: "Retain"

ingress:
  enabled: true
  host: "longhorn.votre-domaine.com"
  tls: true
```

#### Étape 3 : Déployer l'application
1. Cliquez sur **Install**
2. Attendez que tous les pods soient en état **Running**

### Méthode 2 : Installation via kubectl (alternative)

```bash
# Ajouter le repository Helm
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Créer le namespace
kubectl create namespace longhorn-system

# Installer Longhorn
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --set defaultSettings.defaultReplicaCount=3 \
  --set defaultSettings.createDefaultDiskLabeledNodes=true
```

### Vérification de l'installation
```bash
# Vérifier les pods
kubectl get pods -n longhorn-system

# Vérifier les StorageClass
kubectl get storageclass

# Vérifier les nodes Longhorn
kubectl get nodes.longhorn.io -n longhorn-system
```

---

## Partie 4 : Configuration et utilisation (8 minutes)

### Accéder au dashboard Longhorn

#### Via port-forward
```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```
Accédez à http://localhost:8080

#### Via Ingress (si configuré)
Accédez à https://longhorn.votre-domaine.com

### Créer un volume persistent

#### 1. Créer un PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

#### 2. Appliquer la configuration
```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc
```

### Déployer une application test

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: test-container
        image: nginx:latest
        volumeMounts:
        - name: test-volume
          mountPath: /data
        ports:
        - containerPort: 80
      volumes:
      - name: test-volume
        persistentVolumeClaim:
          claimName: test-pvc
```

### Tester la persistance
```bash
# Créer un fichier de test
kubectl exec -it deployment/test-app -- sh -c 'echo "Test Longhorn" > /data/test.txt'

# Vérifier le contenu
kubectl exec -it deployment/test-app -- cat /data/test.txt

# Supprimer le pod et vérifier la persistance
kubectl delete pod -l app=test-app
kubectl exec -it deployment/test-app -- cat /data/test.txt
```

---

## Partie 5 : Opérations avancées (2 minutes)

### Créer un snapshot

#### Via l'interface Longhorn
1. Allez dans **Volume** → Sélectionnez votre volume
2. Cliquez sur **Take Snapshot**
3. Donnez un nom au snapshot

#### Via kubectl
```bash
# Créer un VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: longhorn
  source:
    persistentVolumeClaimName: test-pvc
```

### Configuration des backups

#### Configurer un backup store (exemple S3)
```yaml
# Dans le dashboard Longhorn : Settings → General → Backup Target
s3://votre-bucket@region/path?secret=votre-secret
```

#### Créer un backup
```bash
# Via l'interface : Volume → Create Backup
# Ou via l'API Longhorn
```

---

## Résumé et bonnes pratiques

### Points clés à retenir
- Longhorn fournit un stockage persistant distribué pour Kubernetes
- Installation simple via Rancher ou Helm
- Réplication automatique des données
- Interface graphique pour la gestion
- Support des snapshots et backups

### Bonnes pratiques
- Toujours avoir au moins 3 répliques pour la haute disponibilité
- Configurer des backups réguliers vers un stockage externe
- Surveiller l'espace disque sur les nœuds
- Utiliser des labels pour organiser les volumes
- Tester régulièrement les restaurations

### Ressources supplémentaires
- Documentation officielle : https://longhorn.io/docs/
- GitHub : https://github.com/longhorn/longhorn
- Troubleshooting guide dans la documentation

---

## Questions et discussion
- Quels sont vos cas d'usage pour le stockage persistant ?
- Avez-vous des questions sur la configuration ?
- Besoin d'aide pour des scénarios spécifiques ?
