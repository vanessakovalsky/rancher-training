# Exercice Pratique : Analyser un Workflow GitOps Existant

## 📋 Contexte de l'exercice

Vous êtes consultant DevOps dans une entreprise qui utilise GitOps pour déployer une application web. L'équipe vous demande d'analyser leur workflow actuel pour identifier les points d'amélioration et documenter le processus.

## 🏗️ Architecture du Workflow avec Rancher

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Développeur   │    │   Git Repo      │    │   ArgoCD        │
│                 │    │   (GitLab)      │    │                 │
│  - Code source  │───▶│  - Manifests K8s│───▶│  - Sync auto    │
│  - Tests        │    │  - Pipelines CI │    │  - Monitoring   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
                              ┌─────────────────────────────────────┐
                              │         Rancher Management          │
                              │                                     │
                              │  ┌─────────────┐ ┌─────────────┐   │
                              │  │   Dev       │ │    Prod     │   │
                              │  │  Cluster    │ │   Cluster   │   │
                              │  │             │ │             │   │
                              │  │ - Apps      │ │ - Apps      │   │
                              │  │ - Policies  │ │ - Policies  │   │
                              │  │ - Secrets   │ │ - Secrets   │   │
                              │  └─────────────┘ └─────────────┘   │
                              └─────────────────────────────────────┘
```

## 📁 Structure du Repository avec Rancher

```
e-commerce-app/
├── .gitlab-ci.yml
├── README.md
├── src/
│   ├── frontend/
│   └── backend/
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── ingress.yaml
│       └── prod/
│           ├── kustomization.yaml
│           ├── ingress.yaml
│           └── hpa.yaml
├── rancher/
│   ├── cluster-templates/
│   │   ├── dev-cluster.yaml
│   │   └── prod-cluster.yaml
│   ├── projects/
│   │   ├── e-commerce-dev.yaml
│   │   └── e-commerce-prod.yaml
│   └── apps/
│       ├── app-dev.yaml
│       └── app-prod.yaml
└── argocd/
    ├── app-dev.yaml
    └── app-prod.yaml
```

## 📄 Fichiers de Configuration

### 1. Pipeline CI/CD (.gitlab-ci.yml)

```yaml
stages:
  - test
  - build
  - deploy-manifest

variables:
  DOCKER_REGISTRY: registry.gitlab.com/company/e-commerce
  KUBECONFIG: /tmp/kubeconfig

test:
  stage: test
  script:
    - npm test
    - npm run lint
  only:
    - merge_requests
    - main

build:
  stage: build
  script:
    - docker build -t $DOCKER_REGISTRY:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY:$CI_COMMIT_SHA
  only:
    - main

update-manifest:
  stage: deploy-manifest
  script:
    - sed -i "s|image:.*|image: $DOCKER_REGISTRY:$CI_COMMIT_SHA|g" k8s/base/deployment.yaml
    - sed -i "s|tag:.*|tag: \"$CI_COMMIT_SHA\"|g" rancher/apps/app-prod.yaml
    - git add k8s/base/deployment.yaml rancher/apps/app-prod.yaml
    - git commit -m "Update image to $CI_COMMIT_SHA"
    - git push origin main
  only:
    - main

deploy-to-rancher:
  stage: deploy-manifest
  script:
    - rancher login $RANCHER_URL --token $RANCHER_TOKEN --context c-m-xyz123:p-xyz789
    - rancher app upgrade e-commerce-app-prod --values rancher/apps/app-prod.yaml
  only:
    - main
  when: manual
```

### 2. Deployment Kubernetes (k8s/base/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: e-commerce-app
  labels:
    app: e-commerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: e-commerce
  template:
    metadata:
      labels:
        app: e-commerce
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/company/e-commerce:abc123
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### 3. Application ArgoCD (argocd/app-prod.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: e-commerce-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/company/e-commerce-app.git
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: e-commerce-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 5. Projet Rancher (rancher/projects/e-commerce-prod.yaml)

```yaml
apiVersion: management.cattle.io/v3
kind: Project
metadata:
  name: e-commerce-prod
  namespace: c-m-xyz123
spec:
  displayName: "E-Commerce Production"
  description: "Projet pour l'application e-commerce en production"
  clusterId: "c-m-xyz123"
  resourceQuota:
    limit:
      limitsCpu: "10"
      limitsMemory: "20Gi"
      requestsCpu: "5"
      requestsMemory: "10Gi"
      persistentVolumeClaims: "10"
  namespaceDefaultResourceQuota:
    limit:
      limitsCpu: "2"
      limitsMemory: "4Gi"
      requestsCpu: "1"
      requestsMemory: "2Gi"
  members:
    - accessType: "owner"
      userPrincipalId: "local://u-admin"
    - accessType: "member"
      userPrincipalId: "local://u-devops-team"
```

### 6. Application Rancher (rancher/apps/app-prod.yaml)

```yaml
apiVersion: project.cattle.io/v3
kind: App
metadata:
  name: e-commerce-app-prod
  namespace: p-xyz789
spec:
  description: "E-Commerce Application Production"
  externalId: "catalog://?catalog=helm&template=e-commerce&version=1.2.3"
  projectId: "p-xyz789"
  targetNamespace: "e-commerce-prod"
  values:
    image:
      repository: registry.gitlab.com/company/e-commerce
      tag: "latest"
      pullPolicy: Always
    replicaCount: 5
    service:
      type: ClusterIP
      port: 80
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: "nginx"
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
      hosts:
        - host: ecommerce.company.com
          paths:
            - path: /
              pathType: Prefix
      tls:
        - secretName: ecommerce-tls
          hosts:
            - ecommerce.company.com
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 250m
        memory: 256Mi
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
```

### 7. Cluster Template Rancher (rancher/cluster-templates/prod-cluster.yaml)

```yaml
apiVersion: management.cattle.io/v3
kind: ClusterTemplate
metadata:
  name: production-template
spec:
  displayName: "Production Cluster Template"
  description: "Template standardisé pour les clusters de production"
  members:
    - accessType: "owner"
      userPrincipalId: "local://u-admin"
  defaultRevisionId: "production-template-v1"
  clusterConfig:
    rancherKubernetesEngineConfig:
      kubernetesVersion: "v1.28.8-rancher1-1"
      ignoreDockerVersion: false
      sshAgentAuth: false
      authentication:
        strategy: "x509"
        sans:
          - "ecommerce.company.com"
      network:
        plugin: "canal"
        options:
          flannel_backend_type: "vxlan"
      ingress:
        provider: "nginx"
        options:
          use-forwarded-headers: "true"
      monitoring:
        provider: "metrics-server"
      services:
        etcd:
          backupConfig:
            enabled: true
            intervalHours: 6
            retention: 30
            s3BackupConfig:
              accessKey: "AKIAIOSFODNN7EXAMPLE"
              secretKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
              bucketName: "rancher-etcd-backups"
              region: "eu-west-1"
              endpoint: "s3.eu-west-1.amazonaws.com"
        kubeApi:
          alwaysPullImages: true
          auditLog:
            enabled: true
          eventRateLimit:
            enabled: true
        kubelet:
          failSwapOn: false
          generateServingCertificate: true
  questions:
    - variable: "cluster.name"
      type: "string"
      required: true
      description: "Nom du cluster"
    - variable: "cluster.nodeCount"
      type: "int"
      required: true
      default: 3
      description: "Nombre de nœuds"
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patchesStrategicMerge:
- ingress.yaml

replicas:
- name: e-commerce-app
  count: 5

images:
- name: registry.gitlab.com/company/e-commerce
  newTag: latest
```

## 🔍 Questions d'Analyse

### Partie 1 : Compréhension du Workflow

1. **Décrivez le flux complet** du code source jusqu'au déploiement en production.

2. **Identifiez les outils utilisés** et leur rôle dans le workflow (GitLab CI, ArgoCD, Rancher, Kubernetes).

3. **Quels sont les déclencheurs** de chaque étape du pipeline ?

### Partie 2 : Analyse des Bonnes Pratiques avec Rancher

4. **Sécurité** : Quels aspects de sécurité observez-vous dans ce workflow (RBAC Rancher, gestion des secrets, etc.) ?

5. **Gestion des environnements** : Comment Rancher facilite-t-il la gestion des différents environnements (projets, clusters, quotas) ?

6. **Observabilité** : Quels éléments de monitoring sont intégrés via Rancher et les outils externes ?

7. **Gouvernance** : Comment Rancher applique-t-il les politiques et standards organisationnels ?

### Partie 3 : Identification des Problèmes

8. **Risques identifiés** : Quels problèmes potentiels voyez-vous dans cette configuration Rancher/GitOps ?

9. **Points de défaillance** : Où le workflow pourrait-il échouer (dépendances Rancher, synchronisation, etc.) ?

10. **Gestion des secrets** : Comment les secrets sont-ils gérés entre GitLab, Rancher et les clusters ?

11. **Cohérence multi-clusters** : Comment assurer la cohérence des déploiements sur plusieurs clusters via Rancher ?

### Partie 4 : Propositions d'Amélioration

12. **Optimisations Rancher** : Quelles améliorations proposeriez-vous pour optimiser l'utilisation de Rancher ?

13. **Intégration GitOps** : Comment améliorer l'intégration entre ArgoCD et Rancher ?

14. **Stratégie multi-clusters** : Quelle stratégie recommanderiez-vous pour la gestion multi-clusters ?

15. **Automated Governance** : Comment automatiser davantage la gouvernance via Rancher ?

## 🎯 Livrables Attendus

### Document d'Analyse (30 minutes)

Rédigez un rapport d'analyse contenant :

1. **Schéma du workflow** annoté avec les interactions Rancher/ArgoCD
2. **Analyse des projets et clusters** Rancher avec leurs configurations
3. **Liste des bonnes pratiques** identifiées (GitOps + Rancher)
4. **Catalogue des risques** avec focus sur la centralisation Rancher
5. **Plan d'amélioration** incluant les optimisations Rancher

### Présentation Orale (10 minutes)

Préparez une présentation de 5 minutes + 5 minutes de questions sur :

- Synthèse des points clés (GitOps + Rancher)
- Recommandations pour l'écosystème Rancher
- Stratégie de gouvernance multi-clusters
- Justification des choix d'architecture

## 💡 Indices pour l'Analyse

### Points d'Attention avec Rancher
- Configuration des projets et quotas de resources
- RBAC et gestion des utilisateurs multi-clusters
- Intégration avec les outils GitOps (ArgoCD/Flux)
- Gestion centralisée des policies et compliance
- Monitoring et alerting via Rancher
- Backup et disaster recovery des clusters
- Gestion des certificats et ingress controllers
- Pipeline de déploiement via Rancher CLI/API

### Fonctionnalités Rancher à Explorer
- **Cluster Templates** : Standardisation des configurations
- **Fleet** : GitOps natif pour la gestion multi-clusters
- **OPA Gatekeeper** : Policies de sécurité et compliance
- **Istio Service Mesh** : Gestion du trafic et sécurité
- **Rancher Monitoring** : Stack Prometheus/Grafana intégrée
- **Rancher Logging** : Centralisation des logs
- **Rancher Backup** : Sauvegarde automatisée

### Outils Complémentaires à Considérer avec Rancher
- **Rancher Fleet** pour le GitOps multi-clusters natif
- **Vault** intégré via Rancher pour les secrets
- **Falco** pour la sécurité runtime
- **Longhorn** pour le storage persistant
- **Neuvector** pour la sécurité des containers
- **SUSE Linux Enterprise** pour l'OS optimisé
- **RKE2** pour les clusters de production

## 📚 Ressources Complémentaires

- [Documentation Rancher](https://rancher.com/docs/)
- [Rancher Fleet GitOps](https://fleet.rancher.io/)
- [Documentation ArgoCD](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [GitOps Principles](https://www.weave.works/technologies/gitops/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Rancher Architecture Best Practices](https://rancher.com/docs/rancher/v2.6/en/best-practices/)

---

**Durée estimée** : 3 heures (analyse + présentation)
**Niveau** : Intermédiaire à Avancé
**Prérequis** : Connaissances de base en Kubernetes, Git, concepts DevOps, et notions de Rancher
