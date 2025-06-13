### Objectif
Configurer la réplication des données et tester la récupération après sinistre.

### Partie A : Configuration de la réplication

#### Étape 1 : Vérifier la réplication automatique

```bash
# Lister les volumes avec leurs répliques
kubectl get volumes.longhorn.io -n longhorn-system

# Détail d'un volume spécifique
kubectl describe volume.longhorn.io pvc-xxxxx -n longhorn-system
```

#### Étape 2 : Simuler une panne de nœud

```bash
# Identifier le nœud hébergeant des répliques
kubectl get replicas.longhorn.io -n longhorn-system -o wide

# Drain du nœud (simulation de panne)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

#### Étape 3 : Observer la reconstruction

```bash
# Suivre le statut des volumes
watch kubectl get volumes.longhorn.io -n longhorn-system

# Vérifier la reconstruction via l'UI Longhorn
# Section "Volume" > vérifier le statut "Healthy"
```

### Partie B : Disaster Recovery Volumes

#### Configuration du stockage de sauvegarde

```yaml
# Secret pour l'accès au stockage externe
apiVersion: v1
kind: Secret
metadata:
  name: s3-backup-secret
  namespace: longhorn-system
type: Opaque
data:
  AWS_ACCESS_KEY_ID: <base64-encoded-key>
  AWS_SECRET_ACCESS_KEY: <base64-encoded-secret>
---
# Configuration du backup target
apiVersion: v1
kind: ConfigMap
metadata:
  name: longhorn-backup-target
  namespace: longhorn-system
data:
  backup-target: "s3://votre-bucket@region/longhorn-backups"
  backup-target-credential-secret: "s3-backup-secret"
```

#### Création d'une sauvegarde

```bash
# Via kubectl
kubectl create -f - <<EOF
apiVersion: longhorn.io/v1beta1
kind: BackupVolume
metadata:
  name: backup-mysql-pvc
  namespace: longhorn-system
spec:
  sourceVolumeId: pvc-xxxxx
EOF

# Ou via l'interface web Longhorn
# Volume > Actions > Create Backup
```

#### Restauration depuis sauvegarde

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restored-pvc
  annotations:
    longhorn.io/volume-from-backup: "s3://votre-bucket@region/longhorn-backups?backup=backup-xxxxx"
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

### ✅ Points de validation
- [ ] Réplication fonctionne après panne de nœud
- [ ] Volume reste accessible pendant la reconstruction
- [ ] Sauvegarde créée avec succès
- [ ] Restauration opérationnelle
