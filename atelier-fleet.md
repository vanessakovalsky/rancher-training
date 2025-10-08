# Atelier GitOps avec Fleet - Version Express (60 min)

## Objectifs de l'atelier
- Comprendre GitOps et Fleet
- Configurer un déploiement GitOps basique
- Tester la synchronisation automatique
- Gérer les mises à jour via Git

## Prérequis
- Cluster RKE2 + Rancher opérationnels
- Repository Git préparé
- kubectl configuré

---

## Phase 1 : Setup rapide (10 min)

### Étape 1.1 : Vérification (2 min)
```bash
# Vérifier Fleet
kubectl get pods -n cattle-fleet-system

# Accéder à Rancher → Continuous Delivery
```

### Étape 1.2 : Structure Git (3 min)
```bash
mkdir gitops-demo && cd gitops-demo
mkdir -p {app,fleet-config}
```

### Étape 1.3 : Application exemple (5 min)
```yaml
# app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-svc
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
  type: ClusterIP
```

---

## Phase 2 : Configuration Fleet (15 min)


### Étape 2.1 : Connexion Git dans Rancher (10 min)
1. **Continuous Delivery** → **Git Repos**.
2. Pensez à sélectionner le workspace 'fleet-local' en haut à droite
3. **Add Repository** :
   - **Name** : `demo-gitops`
   - **Repository URL** : `https://github.com/votre-repo/gitops-demo.git`
   - **Branch** : `main`
   - **Paths** : `app`

**Push initial :**
```bash
git add . && git commit -m "Initial setup" && git push origin main
```

---

## Phase 3 : Déploiement et test (20 min)

### Étape 3.1 : Vérification déploiement (5 min)
```bash
# Vérifier les resources Fleet
kubectl get gitrepos -n fleet-local
kubectl get bundles -n fleet-local
kubectl get bundledeployments -A

# Vérifier l'application
kubectl get pods -l app=demo-app
kubectl get svc demo-app-svc
```

### Étape 3.2 : Test GitOps - Mise à jour (10 min)
Modifier le deployment :
```yaml
# app/deployment.yaml - changer replicas
spec:
  replicas: 3  # était 2
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.22  # était 1.21
        env:
        - name: VERSION
          value: "v2.0"
```

```bash
git add . && git commit -m "Scale to 3 replicas + update nginx" && git push
```

### Étape 3.3 : Observer la synchronisation (5 min)
```bash
# Surveiller les changements
watch kubectl get pods -l app=demo-app

# Vérifier Fleet
kubectl get bundles -n fleet-local -w
```

---

## Phase 4 : Multi-environnements express (10 min)

### Étape 4.1 : Configuration par environnement (5 min)
```yaml
# fleet-config/environments.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: Bundle
metadata:
  name: demo-multi-env
spec:
  resources:
  - name: app-manifests
    content: |
      {{ .Files.Get "../app/deployment.yaml" }}
  targets:
  - name: dev
    clusterSelector:
      matchLabels:
        env: dev
    helm:
      values:
        replicas: 1
        image: "nginx:1.21"
        namespace: dev
  - name: prod
    clusterSelector:
      matchLabels:
        env: prod
    helm:
      values:
        replicas: 3
        image: "nginx:1.22"
        namespace: production
```

### Étape 4.2 : Test multi-env (5 min)
```bash
# Labelliser le cluster pour la démo
kubectl label node <node-name> env=dev

# Push et observer
git add . && git commit -m "Multi-env config" && git push
```

---

## Phase 5 : Dépannage rapide (5 min)

### Commandes essentielles
```bash
# Diagnostic Fleet
kubectl describe gitrepo demo-gitops -n fleet-default
kubectl logs -n cattle-fleet-system -l app=fleet-controller --tail=20

# Forcer la synchronisation
kubectl annotate gitrepo demo-gitops -n fleet-default \
  fleet.cattle.io/force-update="$(date)"

# Vérifier les erreurs
kubectl get bundles -A | grep -v Ready
```

### Problèmes courants
- **Bundle pas Ready** → Vérifier les logs Fleet
- **Git non synchronisé** → Vérifier URL/branch/credentials
- **Pods en erreur** → `kubectl describe pod <pod-name>`

---

## Exercice final - Challenge 5 min

**Mission** : Déployer une nouvelle version avec rollback

1. **Mise à jour** : Changer l'image vers `nginx:alpine`
2. **Push** : `git add . && git commit -m "Update to alpine" && git push`
3. **Vérifier** : `kubectl get pods -l app=demo-app`
4. **Rollback** : Revenir à `nginx:1.22` via Git
5. **Confirmer** : Observer la synchronisation automatique

```bash
# Solution rapide
sed -i 's/nginx:alpine/nginx:1.22/g' app/deployment.yaml
git add . && git commit -m "Rollback to 1.22" && git push
```

---

## Récapitulatif et bonnes pratiques

### Ce que vous avez appris (en 60 min !)
✅ Configuration Fleet basique  
✅ Synchronisation Git → Kubernetes  
✅ Mise à jour via GitOps  
✅ Dépannage de base  
✅ Concepts multi-environnements  

### Bonnes pratiques essentielles
1. **Structure Git claire** : Séparer app/config/environments
2. **Commits atomiques** : Un changement = un commit
3. **Monitoring** : Surveiller les bundles Fleet
4. **Rollback via Git** : Historique = source de vérité
5. **Labels clusters** : Pour le targeting multi-env

### Commandes à retenir
```bash
# Fleet
kubectl get gitrepos -A
kubectl get bundles -A
kubectl get bundledeployments -A

# Debug
kubectl describe bundle <name> -n fleet-default
kubectl logs -n cattle-fleet-system -l app=fleet-controller

# GitOps
git add . && git commit -m "message" && git push
```

### Next Steps
- Intégrer CI/CD pipelines
- Secrets management avec Fleet
- Helm charts avancés
- Monitoring avec Prometheus

**🎯 Mission accomplie : GitOps opérationnel en 60 minutes !**
