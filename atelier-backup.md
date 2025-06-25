# Atelier Rancher : Configurer et tester une stratégie de sauvegarde/restauration (45 min)

## Objectifs pédagogiques
- Comprendre les composants essentiels de sauvegarde Rancher
- Configurer Rancher Backup Operator
- Tester une procédure de sauvegarde/restauration
- Valider une stratégie de sauvegarde simple

## Prérequis
- Cluster Rancher fonctionnel (v2.6+)
- Accès administrateur
- kubectl configuré
- Stockage disponible (local ou cloud)

## Durée : 45 minutes

---

## Phase 1 : Concepts essentiels (5 min)

### Types de sauvegardes Rancher

**Rancher Backup Operator** (focus de l'atelier)
- Configuration cluster Rancher
- Projets, utilisateurs, RBAC
- Catalogues et applications
- Ressources Kubernetes managées

**Points clés :**
- Backup = snapshot de la configuration complète
- Restore = restauration sélective ou complète
- Stockage externe recommandé (S3, NFS)

---

## Phase 2 : Installation Rancher Backup Operator (10 min)

### Étape 1 : Installation rapide

```bash
# Via Apps & Marketplace de Rancher
# Ou via kubectl :
kubectl apply -f https://github.com/rancher/backup-restore-operator/releases/latest/download/backup-restore-crd.yaml
kubectl apply -f https://github.com/rancher/backup-restore-operator/releases/latest/download/backup-restore-operator.yaml
```

### Étape 2 : Vérification

```bash
# Vérifier les pods
kubectl get pods -n cattle-resources-system | grep backup

# Vérifier les CRDs
kubectl get crd | grep backup
```

**Résultat attendu :**
```
NAME                                    READY   STATUS    RESTARTS   AGE
rancher-backup-xxx                      1/1     Running   0          30s
```

---

## Phase 3 : Configuration du stockage (5 min)

### Option A : Stockage local (pour démo)

```yaml
# Créer un PV local
apiVersion: v1
kind: PersistentVolume
metadata:
  name: backup-storage
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /opt/rancher-backups
```

### Option B : Stockage S3 (production)

```yaml
# Secret S3
apiVersion: v1
kind: Secret
metadata:
  name: s3-backup-secret
  namespace: cattle-resources-system
type: Opaque
stringData:
  accessKey: "YOUR_ACCESS_KEY"
  secretKey: "YOUR_SECRET_KEY"
```

```bash
kubectl apply -f s3-secret.yaml
```

---

## Phase 4 : Première sauvegarde (10 min)

### Étape 1 : Créer des données de test

```bash
# Créer un projet de test via Rancher UI ou :
kubectl create namespace demo-backup
kubectl label namespace demo-backup field.cattle.io/projectId=demo-project

# Créer quelques ressources
kubectl create deployment nginx-demo --image=nginx -n demo-backup
kubectl create configmap demo-config --from-literal=key=value -n demo-backup
```

### Étape 2 : Créer une sauvegarde manuelle

```yaml
# manual-backup.yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: first-backup
  namespace: cattle-resources-system
spec:
  # Option locale
  storageLocation:
    persistentVolumeClaim:
      claimName: backup-pvc
  
  # Ou option S3
  # storageLocation:
  #   s3:
  #     credentialSecretName: s3-backup-secret
  #     credentialSecretNamespace: cattle-resources-system
  #     bucketName: rancher-backups
  #     region: eu-west-1
```

### Étape 3 : Lancer la sauvegarde

```bash
kubectl apply -f manual-backup.yaml

# Suivre le processus
kubectl get backup first-backup -n cattle-resources-system -w
```

**Vérification :**
```bash
# Vérifier le statut
kubectl describe backup first-backup -n cattle-resources-system

# Vérifier les logs
kubectl logs -l app.kubernetes.io/name=rancher-backup -n cattle-resources-system
```

---

## Phase 5 : Test de restauration (10 min)

### Étape 1 : Simuler une perte de données

```bash
# Supprimer les ressources de test
kubectl delete namespace demo-backup
kubectl delete project demo-project  # Si créé via UI
```

### Étape 2 : Restaurer depuis la sauvegarde

```yaml
# restore-job.yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: restore-first-backup
  namespace: cattle-resources-system
spec:
  backupFilename: first-backup.tar.gz
  storageLocation:
    persistentVolumeClaim:
      claimName: backup-pvc
  
  # Ou pour S3
  # storageLocation:
  #   s3:
  #     credentialSecretName: s3-backup-secret
  #     credentialSecretNamespace: cattle-resources-system
  #     bucketName: rancher-backups
  #     region: eu-west-1
```

### Étape 3 : Lancer la restauration

```bash
kubectl apply -f restore-job.yaml

# Suivre le processus
kubectl get restore restore-first-backup -n cattle-resources-system -w
```

### Étape 4 : Valider la restauration

```bash
# Vérifier que les ressources sont revenues
kubectl get namespace demo-backup
kubectl get deployment nginx-demo -n demo-backup
kubectl get configmap demo-config -n demo-backup
```

**Résultat attendu :** Toutes les ressources sont restaurées à l'identique.

---

## Phase 6 : Stratégie de sauvegarde automatique (5 min)

### Configuration d'une sauvegarde programmée

```yaml
# scheduled-backup.yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: daily-backup
  namespace: cattle-resources-system
spec:
  schedule: "0 2 * * *"        # Tous les jours à 2h
  retentionCount: 7            # Garder 7 sauvegardes
  storageLocation:
    persistentVolumeClaim:
      claimName: backup-pvc
```

```bash
kubectl apply -f scheduled-backup.yaml
```

### Monitoring des sauvegardes

```bash
# Script de vérification rapide
#!/bin/bash
echo "=== État des sauvegardes ==="
kubectl get backups -n cattle-resources-system
echo "=== Dernières restaurations ==="
kubectl get restores -n cattle-resources-system
```

---

## Récapitulatif et bonnes pratiques (5 min)

### Checklist stratégie de sauvegarde

**✅ Configuration de base :**
- [ ] Rancher Backup Operator installé
- [ ] Stockage externe configuré
- [ ] Première sauvegarde testée
- [ ] Procédure de restauration validée

**✅ Production :**
- [ ] Sauvegardes automatiques programmées
- [ ] Stockage redondant (S3 cross-region)
- [ ] Tests de restauration mensuels
- [ ] Documentation procédures d'urgence

### Commandes essentielles à retenir

```bash
# Lister les sauvegardes
kubectl get backups -n cattle-resources-system

# Vérifier une sauvegarde
kubectl describe backup <name> -n cattle-resources-system

# Lancer une restauration
kubectl apply -f restore-job.yaml

# Suivre une restauration
kubectl get restore <name> -n cattle-resources-system -w
```

### Stratégie recommandée

| Environnement | Fréquence | Rétention | Stockage |
|---------------|-----------|-----------|----------|
| Production | Quotidien | 30 jours | S3 cross-region |
| Staging | 2x/semaine | 14 jours | S3 single region |
| Dev | Hebdomadaire | 7 jours | Local/NFS |

### Points d'attention

**🔴 Critiques :**
- Toujours tester les restaurations
- Vérifier l'espace disque disponible
- Protéger les credentials de stockage
- Documenter les procédures d'urgence

**⚠️ Limitations :**
- Rancher Backup ne sauvegarde pas les volumes persistants
- Prévoir une solution complémentaire pour les données applicatives
- Les sauvegardes n'incluent pas les métriques/logs

---

## Exercice pratique final (5 min)

### Défi : Backup et restore complet

**Mission :** Créer une app complète, la sauvegarder, tout supprimer, puis restaurer

```bash
# 1. Créer une app avec plusieurs composants
kubectl create namespace challenge-app
kubectl create deployment web-app --image=nginx -n challenge-app
kubectl create service clusterip web-svc --tcp=80:80 -n challenge-app
kubectl create configmap app-config --from-literal=env=prod -n challenge-app
kubectl create secret generic app-secret --from-literal=key=secret123 -n challenge-app

# 2. Créer une sauvegarde
# (Utiliser la méthode vue précédemment)

# 3. Tout supprimer
kubectl delete namespace challenge-app

# 4. Restaurer et vérifier
# (Utiliser la méthode de restauration)

# 5. Valider que tout fonctionne
kubectl get all -n challenge-app
```

**Temps imparti :** 5 minutes

**Critères de réussite :**
- Toutes les ressources sont restaurées
- Les données des ConfigMaps/Secrets sont intactes
- L'application fonctionne normalement

---

## Ressources pour aller plus loin

### Documentation officielle
- [Rancher Backup Documentation](https://rancher.com/docs/rancher/v2.6/en/backups/)
- [Kubernetes Backup Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/backup/)

### Outils complémentaires
- **Velero** : Sauvegarde des volumes persistants
- **Longhorn** : Snapshots de volumes intégrés
- **Kasten K10** : Solution de sauvegarde enterprise

### Formation continue
- Planifier des tests de restauration mensuels
- Documenter les RTO/RPO de votre organisation
- Former les équipes aux procédures d'urgence
- Mettre en place des alertes sur les échecs de sauvegarde

**🎯 Prochaine étape :** Implémenter cette stratégie sur vos clusters de production !
