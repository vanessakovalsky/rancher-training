### Objectif
Installer Longhorn sur un cluster Rancher et vérifier le déploiement.

### Prérequis techniques
- Cluster Kubernetes opérationnel
- Au moins 3 nœuds worker
- 10GB d'espace libre par nœud
- Packages requis : `open-iscsi`, `util-linux`

### Étapes d'installation

#### Étape 1 : Vérification des prérequis

```bash
# Vérifier les prérequis sur chaque nœud
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/scripts/environment_check.sh | bash

# Installer les dépendances si nécessaire
sudo apt-get update
sudo apt-get install open-iscsi util-linux -y
sudo systemctl enable --now iscsid
```

#### Étape 2 : Installation via Rancher Apps

1. **Accéder au catalogue d'applications Rancher**
   - Naviguer vers "Apps & Marketplace"
   - Sélectionner "Charts"
   - Rechercher "Longhorn"

2. **Configuration de l'installation**
   ```yaml
   # Valeurs de configuration recommandées
   persistence:
     defaultClass: true
     defaultClassReplicaCount: 3
   
   defaultSettings:
     backupstorePollInterval: 300
     createDefaultDiskLabeledNodes: true
     defaultDataLocality: disabled
     replicaSoftAntiAffinity: true
   
   ingress:
     enabled: true
     host: longhorn.votre-domaine.com
   ```

3. **Validation du déploiement**
   ```bash
   # Vérifier les pods Longhorn
   kubectl get pods -n longhorn-system
   
   # Vérifier les nœuds Longhorn
   kubectl get nodes.longhorn.io -n longhorn-system
   
   # Vérifier la StorageClass par défaut
   kubectl get storageclass
   ```

#### Étape 3 : Accès à l'interface web

```bash
# Exposer l'interface (pour test uniquement)
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80

# Ou via l'ingress configuré
curl -I http://longhorn.votre-domaine.com
```

### ✅ Points de validation
- [ ] Tous les pods Longhorn sont en état "Running"
- [ ] StorageClass "longhorn" est disponible
- [ ] Interface web accessible
- [ ] Nœuds détectés avec leur espace de stockage
