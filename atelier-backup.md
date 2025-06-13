### **Objectif**
Configurer et tester une stratégie complète de sauvegarde/restauration.

### **Étape 1 : Installation du Backup Operator**

```bash
# Ajout du repository Helm
helm repo add rancher-charts https://charts.rancher.io
helm repo update

# Installation des CRDs
helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system --create-namespace

# Installation de l'operator
helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system
```

### **Étape 2 : Configuration du stockage**

```yaml
# storage-class.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backup-storage
  namespace: cattle-resources-system
type: Opaque
stringData:
  accessKey: "minioadmin"
  secretKey: "minioadmin"
---
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: daily-backup
  namespace: cattle-resources-system
spec:
  storageLocation:
    s3:
      credentialSecretName: backup-storage
      credentialSecretNamespace: cattle-resources-system
      bucketName: rancher-backups
      endpoint: minio.example.com
      insecureTLSSkipVerify: true
```

### **Étape 3 : Test de restauration**

```bash
# Création d'un backup manuel
kubectl apply -f backup-config.yaml

# Vérification du backup
kubectl get backups -n cattle-resources-system

# Simulation d'une restauration
kubectl apply -f restore-config.yaml
```

### **Validation**
- Vérifier la création des snapshots
- Tester une restauration partielle
- Documenter les temps de restauration
