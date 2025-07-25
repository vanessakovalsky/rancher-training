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
