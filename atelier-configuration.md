# Atelier - Configuration de Rancher Manager

### Objectif
Configurer un environnement Rancher complet avec authentification, RBAC, monitoring et politiques de sécurité

### Environnement
- Cluster RKE2 déployé (Atelier 1)
- Rancher Manager installé
- LDAP/AD de test disponible

### Étapes détaillées

**Etape 0 : Installation de l'UI**

* Installer Helm : https://helm.sh/docs/intro/install/ 
* Suivre les instructions ici : https://ranchermanager.docs.rancher.com/getting-started/quick-start-guides/deploy-rancher-manager/helm-cli#install-rancher-with-helm

**Étape 1: Configuration de l'authentification**

1. Accès au Dashboard Rancher
2. Créer 3 comptes utilisateurs : webdev, devops et lecteur 
3. Test de connexion et validation

**Étape 2: Création de la structure organisationnelle (15 min)**

1. **Création du projet "Production Web":**
   ```yaml
   Project: production-web
   Quotas:
     - CPU Requests: 8 cores
     - Memory Requests: 16Gi
     - Pods: 100
     - PVC: 20
   
   Namespaces:
     - web-frontend
     - web-backend
     - web-database
   ```

**Étape 3: Configuration RBAC (15 min)**

1. **Attribution des rôles globaux:**
   - `devops-engineers` → Cluster Owner
   - `web-developers` → Project Member (production-web)
   - `read-only-users` → Read-Only (global)

2. **Création de rôles personnalisés:**
   ```yaml
   # Rôle "Deployment Manager"
   Permissions:
     - Deployments: Create, Read, Update
     - Services: Create, Read, Update
     - ConfigMaps: Create, Read, Update
     - Secrets: Read (limited)
   ```


**Étape 4: Validation et tests (5 min)**

1. Test d'authentification avec différents utilisateurs
2. Vérification des quotas de projet
3. Validation du monitoring

### Livrables de l'atelier
- Environment Rancher fonctionnel avec authentification
- Structure RBAC complète
- Politiques de sécurité actives
- Documentation des accès et permissions

### Troubleshooting

#### En cas de problèmes pour accèder à l'UI :
* Modifier le service rancher en service de type NodePort, pour cela :
```
kubectl -n cattle-system edit svc rancher
```
* Dans le fichier chercher la ligne avec type: ClusterIp et remplacer par type: NodePort -> enregistrer et quitter le fichier
* Récupérer le port sur lequel est exposé votre service :
```
kubectl -n cattle-system get svc rancher
```
* Cela vous donnera quelque chose de ce type la :
```
rancher                    NodePort    10.43.183.54   <none>        80:31441/TCP,443:31659/TCP   12m
```
* noter le port après 443:
* Enfin nous avons besoin de l'adresse ip du noeud :
```
kubectl get nodes -o wide
```
vous donne un résultat de ce type la
```
 NAME          STATUS   ROLES                       AGE   VERSION          INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
rke2-master   Ready    control-plane,etcd,master   80m   v1.33.5+rke2r1   10.26.90.196   <none>        Ubuntu 22.04.5 LTS   5.15.0-157-generic   containerd://2.1.4-k3s2
```
* noter l'adresse IP dans Internal IP
* pour accéder à votre UI : https://[IP interne du noeud]:[port que vous avez récupérer du service] 
* le certificat est auto signé il y a un avertissement, accepter le risque et continuer
