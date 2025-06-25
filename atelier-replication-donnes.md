# Atelier Rancher - Réplication et Disaster Recovery
**Durée : 60 minutes**

## Objectifs pédagogiques
À la fin de cet atelier, les participants seront capables de :
- Comprendre les stratégies de sauvegarde et de réplication des données dans Rancher
- Configurer Longhorn pour la réplication de volumes
- Mettre en place un plan de disaster recovery
- Tester et valider les procédures de restauration

---

## Prérequis (5 min)
### Environnement technique
- Cluster Rancher fonctionnel avec au moins 3 nœuds
- Longhorn installé comme CSI par défaut
- Accès kubectl configuré
- Helm 3 installé

### Vérification rapide
```bash
# Vérifier l'état du cluster
kubectl get nodes
kubectl get pods -n longhorn-system
rancher --version
```

---

## Module 1 : Fondamentaux de la DR dans Rancher (10 min)

### Concepts clés
- **RTO (Recovery Time Objective)** : Temps maximum acceptable pour restaurer un service
- **RPO (Recovery Point Objective)** : Perte de données maximum acceptable
- **Stratégies** : Sauvegarde, réplication, clustering multi-sites

### Architecture de référence
```
Site Principal (Datacenter A)    Site Secondaire (Datacenter B)
┌─────────────────────────────┐  ┌─────────────────────────────┐
│  Rancher Management Cluster │  │  Rancher Management Cluster │
│  ├── Longhorn Storage      │◄─┤  ├── Longhorn Storage      │
│  ├── Applications          │  │  ├── Applications (Standby) │
│  └── Monitoring           │  │  └── Monitoring             │
└─────────────────────────────┘  └─────────────────────────────┘
```

### Discussion guidée (3 min)
**Question** : Quels sont vos RTO et RPO actuels ? Quels risques identifiez-vous ?

---

## Module 2 : Configuration de Longhorn pour la DR (15 min)

### Étape 1 : Configuration du stockage de sauvegarde
```bash
# Créer un secret pour S3 (exemple avec MinIO)
kubectl create secret generic minio-secret \
  --from-literal=AWS_ACCESS_KEY_ID=minioadmin \
  --from-literal=AWS_SECRET_ACCESS_KEY=minioadmin \
  --from-literal=AWS_ENDPOINTS=http://minio.storage.svc.cluster.local:9000 \
  -n longhorn-system
```

### Étape 2 : Configuration Longhorn via interface web
1. Accéder à l'interface Longhorn : `http://<rancher-url>/k8s/clusters/<cluster-id>/api/v1/namespaces/longhorn-system/services/http:longhorn-frontend:80/proxy/`
2. Aller dans **Settings** → **General**
3. Configurer **Backup Target** :
   ```
   s3://backup-bucket@us-east-1/longhorn-backups
   ```
4. Définir **Backup Target Credential Secret** : `minio-secret`

### Étape 3 : Test de connectivité
```bash
# Vérifier la configuration
kubectl get settings.longhorn.io backup-target -n longhorn-system -o yaml
```

### Manipulation pratique (8 min)
Les participants configurent leur propre cible de sauvegarde et vérifient la connectivité.

---

## Module 3 : Réplication de volumes et sauvegarde (15 min)

### Étape 1 : Création d'une application de test
```yaml
# test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-mysql
  template:
    metadata:
      labels:
        app: test-mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
        - name: MYSQL_DATABASE
          value: "testdb"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: longhorn
```

### Étape 2 : Déploiement et insertion de données
```bash
# Déployer l'application
kubectl apply -f test-app.yaml

# Attendre que le pod soit prêt
kubectl wait --for=condition=ready pod -l app=test-mysql --timeout=300s

# Insérer des données de test
kubectl exec -it deployment/test-mysql -- mysql -uroot -ppassword123 -e "
USE testdb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
SELECT * FROM users;
"
```

### Étape 3 : Configuration des snapshots automatiques
```bash
# Créer une RecurringJob pour snapshots quotidiens
kubectl apply -f - <<EOF
apiVersion: longhorn.io/v1beta1
kind: RecurringJob
metadata:
  name: daily-snapshot
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"
  task: "snapshot"
  groups:
  - default
  retain: 7
  concurrency: 2
EOF
```

### Étape 4 : Sauvegarde manuelle
1. Interface Longhorn → **Volume** → Sélectionner le volume MySQL
2. **Create Backup** → Confirmer
3. Vérifier dans **Backup** que la sauvegarde est créée

### Manipulation pratique (8 min)
Chaque participant crée sa propre application, insère des données et effectue une sauvegarde.

---

## Module 4 : Scénario de disaster recovery (12 min)

### Simulation d'un incident
```bash
# Simuler une perte de données
kubectl delete deployment test-mysql
kubectl delete pvc mysql-pvc
kubectl delete pv $(kubectl get pv | grep mysql-pvc | awk '{print $1}')

# Vérifier que les données sont perdues
kubectl get pvc mysql-pvc
# Résultat attendu : "NotFound"
```

### Restauration depuis sauvegarde

#### Étape 1 : Restauration du volume
1. Interface Longhorn → **Backup**
2. Sélectionner la sauvegarde MySQL → **Restore**
3. Nom du volume : `restored-mysql-volume`
4. Attendre la fin de la restauration

#### Étape 2 : Recréation de l'application
```yaml
# restored-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: restored-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: restored-mysql
  template:
    metadata:
      labels:
        app: restored-mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
        - name: MYSQL_DATABASE
          value: "testdb"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: restored-mysql-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: longhorn
  dataSource:
    name: restored-mysql-volume
    kind: Volume
    apiGroup: longhorn.io
```

#### Étape 3 : Vérification des données
```bash
# Déployer l'application restaurée
kubectl apply -f restored-app.yaml

# Vérifier que les données sont récupérées
kubectl wait --for=condition=ready pod -l app=restored-mysql --timeout=300s
kubectl exec -it deployment/restored-mysql -- mysql -uroot -ppassword123 -e "USE testdb; SELECT * FROM users;"
```

### Manipulation pratique (7 min)
Les participants simulent un incident sur leur application et effectuent une restauration complète.

---

## Module 5 : Plan de disaster recovery et bonnes pratiques (10 min)

### Template de plan DR
```markdown
## Plan de Disaster Recovery - [NOM_APPLICATION]

### Informations critiques
- **RTO cible** : _____ minutes
- **RPO cible** : _____ minutes
- **Volumes critiques** : 
  - Volume 1 : /data/app
  - Volume 2 : /data/db

### Procédure de sauvegarde
1. Fréquence : quotidienne à 02h00
2. Rétention : 7 jours snapshots, 30 jours backups
3. Validation : test de restauration hebdomadaire

### Procédure de restauration
1. Évaluation de l'incident (10 min)
2. Restauration volume (20 min)
3. Redéploiement application (15 min)
4. Tests de validation (10 min)
5. Bascule trafic (5 min)

### Checklist post-incident
- [ ] Données restaurées et vérifiées
- [ ] Applications fonctionnelles
- [ ] Monitoring opérationnel
- [ ] Équipes notifiées
- [ ] Documentation d'incident créée
```

### Bonnes pratiques
1. **Automatisation** : Scripts de restauration automatisés
2. **Tests réguliers** : Validation mensuelle des sauvegardes
3. **Monitoring** : Alertes sur les échecs de sauvegarde
4. **Documentation** : Procédures à jour et accessibles
5. **Formation** : Équipe formée aux procédures DR

### Configuration monitoring
```yaml
# Exemple d'alerte Prometheus pour échecs de backup
groups:
- name: longhorn-backup
  rules:
  - alert: LonghornBackupFailed
    expr: increase(longhorn_backup_state{state="error"}[1h]) > 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "Échec de sauvegarde Longhorn détecté"
      description: "Volume {{ $labels.volume }} - Sauvegarde échouée"
```

### Atelier pratique (5 min)
Chaque participant rédige un plan DR simplifié pour son cas d'usage.

---

## Module 6 : Validation et tests (3 min)

### Checklist de validation
- [ ] Sauvegarde configurée et fonctionnelle
- [ ] Restauration testée avec succès
- [ ] Données intègres après restauration
- [ ] Plan DR documenté
- [ ] Monitoring en place

### Questions de validation
1. Quel est votre RTO après cet atelier ?
2. Comment automatiseriez-vous les tests de restauration ?
3. Quelles métriques surveilleriez-vous ?

---

## Ressources et suite

### Documentation
- [Longhorn Documentation](https://longhorn.io/docs/)
- [Rancher Backup Operator](https://rancher.com/docs/rancher/v2.6/en/backups/)
- [Kubernetes Disaster Recovery](https://kubernetes.io/docs/concepts/cluster-administration/backup/)

### Prochaines étapes recommandées
1. Implémenter la réplication cross-cluster
2. Automatiser les tests de restauration
3. Intégrer avec GitOps (ArgoCD/Flux)
4. Mettre en place une surveillance avancée


