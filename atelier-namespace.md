# Atelier Rancher : Créer une Structure de Namespaces Organisée

**Durée :** 30 minutes  
**Niveau :** Intermédiaire  
**Prérequis :** Accès à Rancher, connaissances de base Kubernetes

## Objectifs pédagogiques

À la fin de cet atelier, vous serez capable de :
- Concevoir une stratégie de nommage cohérente pour les namespaces
- Créer une structure organisée de namespaces dans Rancher
- Appliquer des labels et annotations pour faciliter la gestion
- Configurer des quotas de ressources par namespace
- Mettre en place des politiques de sécurité appropriées

---

## Exercice 1 : Planification de la structure (5 minutes)

### Contexte
Vous devez organiser les namespaces pour une entreprise fictive "TechCorp" qui développe :
- Une application web (frontend + backend + base de données)
- Des services de monitoring
- Des outils de CI/CD
- Un environnement de développement et production

### Tâche 1.1 : Définir la convention de nommage
Créez une convention basée sur le pattern suivant :
```
{environnement}-{équipe}-{application}
```

**Exemples attendus :**
- `prod-webapp-frontend`
- `dev-webapp-backend`
- `prod-infra-monitoring`
- `dev-devops-jenkins`

### Tâche 1.2 : Planifier les labels
Définissez les labels standards à appliquer :
- `environment`: dev/staging/prod
- `team`: webapp/infra/devops
- `component`: frontend/backend/database/monitoring
- `managed-by`: rancher

---

## Exercice 2 : Création des namespaces via Rancher UI (10 minutes)

### Tâche 2.1 : Accéder à la gestion des namespaces
1. Connectez-vous à Rancher
2. Sélectionnez votre cluster
3. Naviguez vers **Cluster → Projects/Namespaces**
4. Cliquez sur **Create Namespace**

### Tâche 2.2 : Créer le namespace de production webapp
1. **Nom :** `prod-webapp-frontend`
2. **Description :** `Frontend de l'application web en production`
3. **Labels :**
   ```yaml
   environment: prod
   team: webapp
   component: frontend
   managed-by: rancher
   ```
4. **Annotations :**
   ```yaml
   contact: webapp-team@techcorp.com
   description: Application frontend React en production
   ```

### Tâche 2.3 : Créer les namespaces suivants
Répétez l'opération pour :
- `prod-webapp-backend`
- `prod-webapp-database`
- `dev-webapp-frontend`
- `dev-webapp-backend`
- `prod-infra-monitoring`

**Astuce :** Utilisez la fonction "Clone" de Rancher pour accélérer la création.

---

## Exercice 3 : Configuration des quotas de ressources (8 minutes)

### Tâche 3.1 : Définir les quotas pour la production
Pour le namespace `prod-webapp-frontend` :

1. Dans Rancher, allez dans le namespace créé
2. Cliquez sur **⋮** → **Edit Config**
3. Activez **Resource Quotas**
4. Configurez :
   ```yaml
   Resource Quotas:
     requests.cpu: "2"
     requests.memory: "4Gi"
     limits.cpu: "4"
     limits.memory: "8Gi"
     pods: "20"
     services: "10"
   ```

### Tâche 3.2 : Quotas pour le développement
Pour `dev-webapp-frontend` :
```yaml
Resource Quotas:
  requests.cpu: "500m"
  requests.memory: "1Gi"
  limits.cpu: "2"
  limits.memory: "4Gi"
  pods: "10"
  services: "5"
```

### Tâche 3.3 : Vérification
1. Vérifiez que les quotas sont appliqués via **Cluster → Resource Quotas**
2. Notez la différence entre les environnements de dev et prod

---

## Exercice 4 : Mise en place de Network Policies (5 minutes)

### Tâche 4.1 : Isolation des environnements
1. Naviguez vers **Policy → Network Policies**
2. Créez une policy pour isoler la production :

**Nom :** `prod-isolation`
**Namespace :** `prod-webapp-frontend`
**Configuration :**
```yaml
# Autoriser seulement le trafic depuis les autres namespaces prod
podSelector: {}
policyTypes:
  - Ingress
ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: prod
```

### Tâche 4.2 : Communication inter-services
Créez une policy pour permettre la communication backend-database :

**Nom :** `webapp-backend-to-db`
**Namespace :** `prod-webapp-database`
```yaml
podSelector:
  matchLabels:
    app: database
ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: prod
          team: webapp
    - podSelector:
        matchLabels:
          app: backend
```

---

## Exercice 5 : Validation et bonnes pratiques (2 minutes)

### Tâche 5.1 : Vérification de la structure
1. Allez dans **Cluster → Projects/Namespaces**
2. Vérifiez que tous les namespaces sont créés avec les bons labels
3. Testez le filtre par label `environment=prod` -> pas disponible dans l'UI le faire en CLI :
```
kubectl get ns -l environnement=prod
```

### Tâche 5.2 : Documentation
Créez une ConfigMap avec la documentation de votre structure :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: namespace-documentation
  namespace: default
data:
  naming-convention: |
    Pattern: {environnement}-{équipe}-{application}
    Environnements: dev, staging, prod
    Équipes: webapp, infra, devops
  contact-info: |
    webapp-team: webapp-team@techcorp.com
    infra-team: infra-team@techcorp.com
    devops-team: devops-team@techcorp.com
```

---

## Points clés à retenir

### ✅ Bonnes pratiques appliquées
- **Convention de nommage cohérente** : Facilite la navigation et l'automatisation
- **Labels standardisés** : Permettent le filtrage et la gestion par lots
- **Quotas différenciés** : Protègent les ressources selon l'environnement
- **Isolation réseau** : Sécurise les communications inter-services
- **Documentation** : Facilite la maintenance et l'onboarding

### 🔧 Outils Rancher utilisés
- Interface de gestion des namespaces
- Système de labels et annotations
- Configuration des quotas de ressources
- Gestion des Network Policies
- Filtrage et recherche

### 📈 Avantages de cette organisation
- **Sécurité** : Isolation entre environnements
- **Gouvernance** : Contrôle des ressources
- **Maintenance** : Structure claire et documentée
- **Automatisation** : Compatible avec GitOps et CI/CD
- **Collaboration** : Responsabilités d'équipe claires

---

## Exercices d'approfondissement (optionnel)

Si vous avez terminé en avance, essayez :

1. **Créer des Projects Rancher** pour regrouper les namespaces par équipe
2. **Configurer des PodSecurityPolicies** spécifiques par environnement
3. **Mettre en place des LimitRanges** pour des contraintes par pod
4. **Tester le déploiement** d'une application simple dans chaque namespace

---

## Ressources supplémentaires

- [Documentation Rancher - Namespaces](https://rancher.com/docs/rancher/v2.6/en/cluster-admin/namespaces/)
- [Kubernetes - Namespace Best Practices](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Network Policies Guide](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
