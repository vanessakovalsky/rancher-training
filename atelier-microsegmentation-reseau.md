# Atelier Rancher : Implémenter la Microsegmentation Réseau
**Durée : 30 minutes | Niveau : Intermédiaire**

## 🎯 Objectifs pédagogiques
À la fin de cet atelier, vous saurez :
- Comprendre les concepts de microsegmentation dans Kubernetes
- Configurer des Network Policies avec Rancher
- Implémenter des règles de trafic granulaires
- Tester et valider la segmentation réseau

## 📋 Prérequis
- Cluster Kubernetes fonctionnel avec Rancher
- CNI compatible Network Policies (Canal (par défaut), Calico, Cilium, ou Weave)
- Accès administrateur au cluster
- kubectl configuré

---

## ⏱️ Phase 1 : Configuration initiale (5 minutes)

### Étape 1.1 : Vérifier l'environnement
```bash
# Vérifier le CNI installé
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave|canal)"

# Créer les namespaces de démonstration
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
```

### Étape 1.2 : Déployer les applications de test
```yaml
# frontend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend
  labels:
    app: frontend
    tier: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f frontend-app.yaml
```

---

## ⏱️ Phase 2 : Microsegmentation de base (8 minutes)

### Étape 2.1 : Créer une politique de déni par défaut
```yaml
# default-deny-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Étape 2.2 : Appliquer via Rancher UI
1. Accéder à **Cluster Explorer** → **Network** → **Network Policies**
2. Cliquer sur **Create from YAML**
3. Coller la politique et appliquer
4. Répéter pour les namespaces `backend` et `database`

### Étape 2.3 : Tester l'isolation
```bash
# Déployer un pod de test
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Dans le pod, tester la connectivité (doit échouer)
wget -qO- --timeout=3 frontend-service.frontend.svc.cluster.local
```

---

## ⏱️ Phase 3 : Règles granulaires (10 minutes)

### Étape 3.1 : Autoriser le trafic web entrant
```yaml
# allow-web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### Étape 3.2 : Communication inter-tiers
```yaml
# frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Étape 3.3 : Règles de sortie pour DNS
```yaml
# allow-dns-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 8080
```

---

## ⏱️ Phase 4 : Gestion avancée avec Rancher (5 minutes)

### Étape 4.1 : Utiliser l'interface Rancher
1. **Projects/Namespaces** → Sélectionner le projet
2. **Resources** → **Network Policies**
3. **Add Policy** → Utiliser le wizard graphique

### Étape 4.2 : Template de politique réutilisable
```yaml
# rancher-network-template.yaml
apiVersion: management.cattle.io/v3
kind: ProjectNetworkPolicy
metadata:
  name: web-tier-template
spec:
  projectId: "c-m-xxxxx:p-xxxxx"
  description: "Template pour applications web"
  networkPolicy:
    podSelector:
      matchLabels:
        tier: web
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - namespaceSelector: {}
      ports:
      - protocol: TCP
        port: 80
```

---

## ⏱️ Phase 5 : Validation et troubleshooting (2 minutes)

### Étape 5.1 : Commandes de diagnostic
```bash
# Lister toutes les Network Policies
kubectl get networkpolicies --all-namespaces

# Détails d'une politique
kubectl describe networkpolicy default-deny-all -n frontend

# Logs CNI (exemple avec Calico)
kubectl logs -n kube-system -l k8s-app=calico-node --tail=20
```

### Étape 5.2 : Tests de connectivité
```bash
# Script de test automatisé
#!/bin/bash
NAMESPACES=("frontend" "backend" "database")

for ns in "${NAMESPACES[@]}"; do
  echo "Testing connectivity from $ns..."
  kubectl run test-$ns --image=busybox --rm -it --restart=Never -n $ns \
    -- wget -qO- --timeout=3 frontend-service.frontend.svc.cluster.local
done
```

---

## 📊 Bonnes pratiques et recommandations

### Stratégie de déploiement
1. **Commencer par l'observation** : Analyser le trafic existant
2. **Déploiement progressif** : Une application à la fois
3. **Mode Warning** : Tester avant de bloquer
4. **Documentation** : Maintenir un inventaire des règles

### Patterns courants
- **Déni par défaut** : Base de toute stratégie de microsegmentation
- **Autorisation explicite** : Seul le trafic nécessaire est autorisé
- **Séparation par environnement** : Dev/Test/Prod isolés
- **Principe du moindre privilège** : Permissions minimales

### Surveillance et monitoring
```bash
# Métriques Network Policies (avec Prometheus)
kubectl get --raw /metrics | grep networkpolicy

# Alertes recommandées
# - Politique bloquant du trafic inattendu
# - Tentatives de connexion refusées
# - Modifications non autorisées des politiques
```

---

## 🎯 Points clés à retenir

1. **La microsegmentation nécessite un CNI compatible** (Calico, Cilium, Weave)
2. **Toujours commencer par une politique de déni par défaut**
3. **Tester exhaustivement avant la production**
4. **Rancher simplifie la gestion via son interface graphique**
5. **La documentation des flux est essentielle**
6. **Prévoir des procédures de rollback rapide**

## 📚 Ressources complémentaires
- [Documentation officielle Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Guide Rancher Network Policies](https://rancher.com/docs/rancher/v2.x/en/k8s-in-rancher/network-policies/)
- [Calico Network Policy Tutorial](https://docs.projectcalico.org/security/tutorials/kubernetes-policy-basic)

---
**💡 Conseil formateur :** Encouragez les participants à expérimenter avec différents scénarios et à adapter les règles à leurs cas d'usage spécifiques.
